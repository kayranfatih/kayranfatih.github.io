---
layout: post
title: Yük Dengeleyiciler ve Sticky Oturum Cookieleri
categories:
- blog
---

Uygulama/Veritabanı sunucularında kullanıma bağlı olarak artan kaynak ihtiyacı, bir sunucu üzerinde daha fazla kaynak bulundurmak yerine yükü birden fazla sunucuya paylaştırma yöntemleri kullanılarak çözüme ulaşmaktadır. Horizontal scaling olarak adlandırılan bu yaklaşım devasa işlem gücü gerektiren web uygulamalarında oldukça yaygın kullanılmaktadır. Veritabanı tarafında Oracle’ın geliştirdiği RAC mimarisi horizontal scaling’in üstesinden gelirken web sunucuları tarafında load balancing yöntemi kullanılmaktadır. Bu noktada yük dengeleyiciler(loadbalancer) olarak adlandırılan sunucular devreye girmektedir.
{: style="text-align: justify;"}

---

Web sunucularının önünde konuşlandırılan loadbalancerlar, requestleri web sunucularından bir önceki hop’ta karşılar(çoğu zaman SSL’i de çözer) ve web sunucularının o anki kaynak kullanımı durumlarına göre uygun olan sunucuya yönlendirir. Bu yönlendirme esnasında loadbalancer bir proxy sunucu gibi çalışır ve client, loadbalancer ile el sıkışmayı tamamladıktan sonra requestleri -loadbalancer üzerinden- web sunucularına gönderebilir. Bu esnada web uygulamaları -layer 3 seviyesinde- loadbalancer ile konuşacağından dolayı IP katmanında göreceği IP adresi loadbalancerın web sunucularına bakan ayağındaki IP adresi olacaktır. Log yönetimi, uygulama üzerinden yapılan IP kontrolleri gibi durumlarda clientların gerçek IP adresi gerektiğinden dolayı bu durumun aşılması için ek konfigürasyona gerek vardır. Layer 4 veya layer 7 seviyesinde client IP’lerinin web uygulamasına iletilmesi ile gerçek IP problemi ortadan kalkmaktadır fakat ortaya bu IP’lerin spoof edilme olasılığı çıkmaktadır. Mehmet İnce, bloğunda yer verdiği yazıda bu durumu F5 loadbalancer ve XFF üzerinden oldukça detaylı açıklamıştır.
{: style="text-align: justify;"}

---

## Citrix Netscaler Loadbalancer ve Client IP

F5 gibi loadbalancer konusunda kafaya oynayan Citrix firması tarafından geliştirilen Netscaler loadbalancerlarında XFF headerı kullanılarak yapılabileceği gibi custom bir headerın tanımlanması ile de gerçek client IP problemi aşılabilmektedir. Bu makalede Netscaler için custom headerın nasıl tanımlandığı anlatılmıştır. Header adı tanımlama aşamasında custom olarak verildiğinden -XFF gibi generic bir header değil- dolayı XFF’ten daha güvenli bir yöntem olarak görülmekte fakat headerın doğru tahmin edilebilme ihtimali ile IP spoof gerçekleştirilebilmektedir. Bu sorun -Mehmet İnce’nin bahsettiği gibi- gelen requestler arasından bu headerın loadbalancer üzerinden yazılan bir kuralla ayıklanması ve layer 3’ten aldığı IP adresini custom olarak tanımlanan headera basması ile aşılabilmektedir.
{: style="text-align: justify;"}

---

## Sabit Loadbalancer Oturum Cookieleri

Loadbalancer kullanan sistemlerde karşılaşılan bir başka sorun ise web sunucusunda tutulan session değerlerinin akıbetinin ne olacağıdır. Clientlar bir uygulamada authentication sürecini atlattıktan sonra sunucu tarafında bir session tahsis edilir(aynı zamanda clienta bir cookie gönderilir). Şuanki durumda web sunucusunun tahsis ettiği cookie değeri o sunucuya özel olduğundan dolayı şu senaryo oluşabilir : Session değerine sahip web sunucusunun yükünün artmasından dolayı loadbalancer başka bir sunucuya yönlendirme yapabilir ve session değerinin kaybolmasına neden olabilir. Böyle bir durumla karşılaşan bir client authenticate olduğu bir uygulamadan bir anda login ekranına tekrardan düşecek ve kullanıcı deneyiminin(UX/User Experience) olumsuz etkilenmesine neden olacaktır.
{: style="text-align: justify;"}

Tam bu noktada loadbalancerların web sunucuları tarafında tutulan session, cache değerlerini koruması için farklı çözümler geliştirilmiştir.  Loadbalancer arkasında bulunan web sunucularının session, cache bilgilerinin ortak bir alanda tutulması ile loadbalancer farklı sunuculara yönlendirme yapsa dahi session değerleri korunabilmektedir. Sessionların bir veritabanına depolanması veya memcached gibi yaklaşımlar bu yöntem kullanılarak geliştirilen çözümler olarak nitelendirilebilir.
{: style="text-align: justify;"}

Diğer yöntem ise loadbalancerın ilk yönlendirmeyi gerçekletirdiği web sunucusunu oturum bitene kadar hatırlamasıdır. Bu şekilde client oturum süresince tek bir web sunucusu ile konuşacak(yani loadbalancer her zaman o client için aynı web sunucusuna yönlendirme yapacak) ve session değeri korunacaktır. Bu durumda loadbalancerın clientın hangi client olduğunu tanıması gerekmektedir ve ortaya loadbalancer tarafından eklenmiş cookie değeri çıkmaktadır. Bu cookie değeride sticky session cookie olarak adlandırılmaktadır. Aşağıda set-cookie ile setlenmiş bir Netscaler sticky cookie(NSC_xxx) değeri görünmektedir.
{: style="text-align: justify;"}

>HTTP / 1.1 200 OK
Date: Tue, 14 Feb 2017 21:42:27 GMT
Pragma: no-cach
Vary: Accept-Encoding
Content-Type: text/html; charset=utf-8
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: accept, authorization
Cache-Control: max-age=0, no-cache
Connection: keep-alive
Accept-Ranges: bytes
Set-Cookie: NSC_xxx.nzofu.dpn_dt_efofnf_2013=ffffffffc3a0346345525d5f4f58455e445a4a423660;path=/;httponly

Sticky session cookieler ile yapılan yönlendirmeler loadbalancerın uygulama sunucularındaki yük kontrolünü sadece oturum başlangıç anında gerçekleştirmesine, yönlendirme işlemini bir kere yapmasına neden olmaktadır. Bir sunucunun çökmesinde sunucuya bağlı olan erişimin kesilmesine, persistent cookie kullanan(facebook gibi authenticate olduktan sonra sürekli hesabın açık kalması) uygulamalar için yük durumunun kontrolünün olmamasına neden olmaktadır.
{: style="text-align: justify;"}

---

## Citrix Netscaler Loadbalancer

Sticky session cookilerin aynı zamanda encrypt edilmiş bir değerdir ve decrypt edilmesi mümkün olmaktadır. F5 ve Netscaler sticky cookieleri bu şekilde decrypt edilebilmekte ve loadbalancer arkasındaki web sunucularının iç IP adresleri deşifre edilebilmektedir. Github üzerinden erişebileceğiniz Netscaler Cookie Decryptor scripti ile örnek olarak yukarıda verilmiş netscaler cookiesini decrypt edilmiş hali aşağıdaki gibidir.(Domain bölümü sansürlenmiştir)
{: style="text-align: justify;"}

>kayran@komutan:~/Desktop# python nsccookiedecrypt.py NSC_xxx.nzofu.dpn_dt_efofnf_2013=ffffffffc3a0346345525d5f4f58455e445a4a423660
vServer Name=www.xxxxxx.com_cs_deneme_2013
vServer IP=192.168.42.114
vServer Port=80

---

## Öneriler

Loadbalancer kullanılan bir sistemde best-practice olarak sticky cookie kullanılması uygun bulunmuyor fakat farklı sunuculara uygulamaların farklı modüllerini deploy eden yapılarda sticky cookie kullanımına mecbur kalınıyor. Uygulamayı kullanılan virtual IP bloklarına göre sunucular üzerinde aynı şekilde deploy edip, memcached gibi bir yaklaşım ile sticky cookie kullanımına gerek kalınmıyor. Sticky cookie kullanımının zorunlu olduğu durumlarda ise loadbalacerlarda cookielerin ek bir encryption methoduyla encrypt edilebilir. Encrpytion özelliği birçok loadbalancerda varsayılan olarak bulunmaktadır. Ayrıca sticky cookie'ler için varsayılan(Netscaler'da NSC_XXX...) isimler yerine farklı isimler kullanılabilir.
{: style="text-align: justify;"}

---

## Kaynakça

[Mehmet İnce - Yük Dengeleyiciler ve Gerçek IP Adresi Karmaşası](https://www.mehmetince.net/yuk-dengeleyiciler-ve-gercek-ip-adresi-karmasasi/)

[Netscaler Making Sense of the Cookie Part-1](https://itgeekchronicles.co.uk/2012/01/03/netscaler-making-sense-of-the-cookie-part-1/)

[Netscaler Making Sense of the Cookie Part-2](https://itgeekchronicles.co.uk/2012/01/06/netscalers-making-sense-of-the-cookie-part-2/)

[Netscaler Making Sense of the Cookie Final](https://itgeekchronicles.co.uk/2012/01/23/netscalers-making-sense-of-the-cookie-the-finale/)

[Netscaler Cookie Decryptor](https://github.com/catalyst256/Netscaler-Cookie-Decryptor)
