# **SSRF NEDİR? (Server Side Request Forgery)**

Türkçesi sunucu taraflı istek sahteciliği olan bu güvenlik açıklığı,  saldırgana güvenlik açığı bulunan sunucudan istek oluşturması veya  buradan gelen istekleri kontrol edebilmesi için web uygulamasındaki bir  parametreyi değiştirmesine izin verir. Başka bir değişle saldırganın  doğrudan yapamadığı işlemleri sunucuya yaptırarak saldırganın sunucuyu  sömürmesi de diyebiliriz.

Web uygulamaları, genellikle yazılım güncellemeleri gibi uzak  kaynakları almak veya bir URL’den ya da diğer web uygulamalarından  içeriye veri aktarmak için yapılan sunucular arası istekleri  tetikleyebilir. Bu tür sunucular arası istekler, genelde güvenli olsa da  implementasyon doğru yapılmadığı takdirde SSRF zafiyetine yol  açabilirler.

Bir web uygulamasındaki bilgiler, başka bir web sitesinden gelen  harici bir kaynaktan alınmak zorundaysa, sunucu tarafı istekleri kaynağı  alıp web uygulamasına dahil etmek için kullanılır.

## SSRF Zafiyetinin Kullanımı

![SSRF Zafiyet Olay Akışı](images/Turkce-ssrf/ssrf.png)

1. SSRF zafiyetini kötüye kullanan bir isteği güvenlik açığı bulunan web sunucusuna göndermelidir.
2. Web sunucusu, güvenlik duvarının arkasındaki kurbanın sunucusuna bir istek yapar.
3. Kurbanın sunucusu veriyle cevap verir.
4. Eğer SSRF açığı buna izin veriyorsa, veri saldırgana geri gönderilir.

## SSRF Zafiyetinin Yansımaları

### 1. Etkilenen sunucuya duyulan güvenin kötüye kullanımı

En iyi yöntem olarak, saldırı yüzeyini mümkün olduğu kadar küçük  tutmak her zaman daha iyidir, bu nedenle belirli portlara veya eylemlere  erişim genellikle, sadece whitelist’e alınmış makinelerle  sınırlandırılmaktadır. Aslında sunucular, verileri kolayca paylaşmak ve  yönetimsel görevlere izin vermek için genellikle diğer makinelerle güven  ilişkisine sahiptir.

Örneğin ağ seviyesinde güven şu anlama gelir: Bir güvenlik duvarı,  eğer erişim talep eden makine aynı yerel ağda bulunuyorsa veya makinenin  IP adresine güveniliyorsa, sadece belirli portlara erişim izni verir.

Yazılım düzeyinde güven şu şekilde olabilir: IP adresi 127.0.0.1  olduğu sürece veya yerel ağın içindeyse, bazı yönetimsel görevler için  kimlik doğrulama gerekli değildir. Bu tür bir güven, bir saldırganın  parolayı bildiği halde yerel ağa erişim olmadan giriş yapamamasını  sağlamak için ek bir güvenlik önlemi olarak da kullanılabilir.

Saldırgan, SSRF’ten yararlanarak yukarıdaki kısıtlamaları atlatabilir  ve örneğin; etkilenen makineye güvenen diğer sunucuları sorgulayabilir  veya dış ağdan erişilemeyen bazı portlarla etkileşim kurmak için kötü  amaçlı isteklerde bulunabilir. Saldırgan, bu şekilde dışarıdan yapılması  mümkün olmayan kötü amaçlı eylemleri sunucunun kendisini kullanarak  sunucu üzerinde gerçekleştirebilir

### 2. Yerel veya Harici Ağların Taranması

Saldırganlar, SSRF güvenlik açığından yararlanarak, güvenlik açığı  bulunan sunucunun bağlı olduğu yerel veya harici ağları tarayabilir.  Saldırganlar genellikle bir sayfanın yüklenme süresini, sayfadaki hata  mesajını veya hedefledikleri araştırmadan cevap alıp alamadıklarını  belirlemek ve test edilen portun açık olup olmadığını doğrulamak için  sorguladıkları banner bilgisini kullanırlar.

### Bir Ağın SSRF Zafiyeti Kullanılarak Nasıl Taranacağına Dair Örnek

Uzak kaynaktan jpeg formatında resim çağırmanıza izin veren ve bu  işlem sırasında resimlerin boyutlarını belirleyebilen bir web servisi  hayal edin.

Önlem olarak, servis uzak kaynaktan gelen istekte "**image/jpeg**" değerine sahip bir **Content-Type**  headerı olup olmadığını denetler. Eğer Content-Type başlık alanı değeri  “image/jpeg” değilse veya içerik tipi farklıysa, dosyanın jpeg  olmadığını varsayarak “Sadece jpeg imajları kabul edilmektedir” şeklinde  bir hata döndürür. Bunun haricinde, uzaktaki kaynağa bağlanmada bir  problem varsa, "**Resim bulunamadı!**" hatası döndürülür.

Saldırganların host’un ayakta olup olmadığını anlamak için web  servisinin yanıtını etkileyen nasıl farklı inputlar kullanabildiğine  bakalım.

`https://victim.com/image.php?url=https://example.com/picture.jpg`  şeklinde istekte bulunursak cevap olarak resmin yüksekliğini ve  genişliğini alırız. Ki bu, Content-Type header’ı doğru olan geçerli bir  resim olduğu anlamına gelir.

Şimdi farklı bir istekte bulunmayı deneyelim ve sunucuya yaptığımız istekte resim yerine bir HTML dosyası gönderelim: `https://victim.com/image.php?url=https://example.com/index.html` Bu durumda yüklenen sayfanın Content-Type header’ı “text/html” olduğundan servisin cevabı “**Sadece jpeg imajları kabul edilmektedir**” olur.

Şimdi geçersiz bir URL ile istekte bulunalım; `https://victim.com/image.php?url=https://example.invalid/` Servis “**Resim bulunamadı**” hatasının döndürür. Bunun anlamı uzak kaynak alınırken bir hata oluştuğudur.

Artık uygulamanın farklı girdiler için nasıl davrandığını  bildiğimizden kötüye kullanmayı deneyebiliriz. Yanlış veya eksik  Content-Type header’ı olan geçerli bir URL'in “Sadece jpeg imajları  kabul edilmektedir” hatasını döndürdüğünü biliyoruz. Bunun anlamı  aşağıdaki isteği gönderirsek;

`https://victim.com/image.php?url=127.0.0.1:3306`

Ve “Resim bulunamadı” hatası alırsak, bunun anlamı 127.0.0.1 port  3306’da yanıt yok demektir. Eğer “Sadece jpeg imajları kabul  edilmektedir” hatasını alırsak sunucudan gelen bir yanıtın olduğunu  dolayısıyla söz konusu port üzerinde bir servisin çalıştığını  anlayabiliriz. Bundan dolayı bu metodu, tam bir tarama yapma amacıyla  farklı dahili IP adreslerini ve portlarını sorgulamak için  kullanabiliriz.

### 3. Sunucudan Dosyaların Okunması

Uzak bir kaynağın içeriği doğrudan bir sayfaya yansıtıldığında,  saldırganların dosyaların içeriğini okuma olasılığı vardır. Örnek  olarak, verilen bir url'den tüm resimleri kaldıran ve metni  biçimlendiren bir web servisi düşünün. İlk önce belirli bir url'nin  yanıt gövdesini elde ederek çalışır, daha sonra biçimlendirmeyi uygular.

Eğer **http://** veya **https://** yerine **file://**  kullanırsak yerel dosya sisteminden dosyaları okuyabiliriz. Örneğin  file:///etc/passwd kullanırsak unix sistemlerindeki passwd dosyasının  içeriğini sayfaya yazdırabiliriz. Aynı teknik zafiyet bulunan web  uygulamasının kaynak kodunu görmek için de kullanılabilir.

### **Siteler arası betik çalıştırma:**

SSRF  açıklığını kullanarak reflected XSS atakları düzenlenebilir. Bu ataklar  sayesinde kullanıcıların cookie bilgileri çalınabilir, başka sayfaya  yönlendirme, hatta BeeF kullanarak browser üzerinde komut bile  çalıştırılabilir.
XSS atağı düzenlerken kullanacağımız zararlı javascript kodu içeren dosyayı da yine uzak sunucudan çekebiliriz.
[http://webuygulamasi.com/?url=http://brutelogic.com.br/poc.svg](http://webuygulamasi.com/?url=http://brutelogic.com.br/poc.svg)

## SSRF Zafiyetinin Etkileri

* Siteler arası betik çalıştırma, (SSRF to Reflected XSS)
* Server içerisinde uzaktan kod çalıştırma, (SSRF to RCE)
* Sunucuda kritik dosyaların içeriğinin okunması, (SSRF to LFI)
* URL şemaları kullanarak sunucuya işlem yaptırılması, (ftp://, dict://, http://, gopher://, file:// vb.)
* Cloud servislerin meta-data bilgilerinin çekilmesi,
* Whitelist ve DNS çözümlemelerinin aşılması gibi faaliyetler de bulunulabilir.
* Daha fazla keşif yapmanıza izin veren Gopher gibi bazı protokollerle etkileşime izin verir.
* Bir reverse proxy’nin arkasındaki sunucuların IP adreslerini tespit edebilir.
* Bu ana makinelerde çalışan hizmetleri numaralandırmak ve saldırmak
* Host-Based tabanlı kimlik doğrulama hizmetlerinden faydalanarak authentication bypass
* Status Pages (Durum Sayfaları)'i görüntüleyebilir ve API'larla kurulan  etkileşimde, zafiyetin bulunduğu web sunucusunun yerine geçebilir.
* Sunucuya bağlı olan yerel ağı tarayabilir.
* Sunucuda bulunan dosyaları okuyabilir.
* Zafiyeti barındıran sunucu ile diğerleri arasındaki güven ilişkisini kötüye kullanabilir.
* IP kontrolünü atlatabilir.

Saldırganlar, SSRF zafiyetinden yararlanarak bir dizi tehlikeli eylemi gerçekleştirebilirler. Bunların bazıları çok ciddi sonuçlar doğurabilmektedir. Fakat bu eylemlerin potansiyel sonuçları, genellikle web uygulamasının uzak kaynaklardan gelen istekleri nasıl işlediğine bağlıdır.

## **SSRF Zafiyeti Önleme Metotları**

### **DNS Çözümleme ve Whitelist Oluşturma:**

SSRF  tehdini önlemenin en sağlam yolu, uygulamanızın erişmesi gereken DNS  adını veya IP adresini whitelist yöntemiyle erişime açmaktır. Whitelist  yaklaşımı size uymuyorsa mecburen bir blacklist oluşturup güvenmediğiniz  IP adreslerini tutabilirsiniz. Burada önemli olan kullanıcı erişimini  doğrulamaktır.
Blacklist yöntemi için sayısız ihtimali azaltma yolu  olarak da düşünebiliriz. SSRF için evrensel bir önleme metodu olmadığı  için mecburiyet halinde kullanılmalıdır.

### **Kullanılmayan URL Şemalarını Devre Dışı Bırakma:**

Uygulamanız  istekte bulunmak için yalnızca HTTP veya HTTPS kullanıyorsa, yalnızca  bu URL şemalarına izin verin. Kullanılmayan URL şemalarını devre dışı  bırakırsanız, saldırgan, `file://, dict://, ftp:// ve gopher://` gibi  tehlikeli şemalar kullanarak istekte bulunmak için web uygulamasını  kullanamaz.

### **İç Ağda Bulunan Servislere Erişim İçin Kimlik Doğrulama:**

Varsayılan  olarak, Memcached, Redis, Elasticsearch ve MongoDB gibi servislerin  onaylanması gerekmez. Saldırgan, bu hizmetlere kimlik doğrulaması  yapmadan erişebilir. Bu nedenle yerel ağdaki hizmetler için mümkün  olduğunca kimlik doğrulamasını etkinleştirmek gerekmektedir.

### **Sunucu Taraflı Yanıt Kontrolü (Response Handling):**

Saldırganların **SSRF**  ile sunucudan sömürü amaçlı aldığı yanıtlar normalden farklı olacaktır.  Sunucudan gönderilen yanıtların veya isteklerin kontrol edilip çıktığı  gibi iletilmemesi gerekmektedir. Aksi takdirde **SSRF** ile verilen yanıtlar saldırganın isteğine göre kontrolsüz şekilde gidecektir.


## Referans ve Kaynakça:

https://www.netsparker.com.tr/blog/web-guvenligi/ssrf-nedir-ssrf-nasil-onlenir/

https://www.gaissecurity.com/blog/ssrf-server-side-request-forgery

https://portswigger.net/web-security/ssrf

https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery#gopher
