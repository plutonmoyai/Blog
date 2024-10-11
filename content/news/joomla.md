# Joomla Nedir?

Joomla, PHP ve MySQL ile MVC olarak geliştrilimiş web siteleri ve çevrimiçi uygulamalar oluşturmak için kullanılan ücretsiz ve açık kaynaklı bir İçerik Yönetim Sistemidir (CMS). Tüm web sitelerinin yaklaşık %2'si Joomla kullanmaktadır, bu da onu dünya çapında milyonlarca dağıtımla en popüler CMS'lerden biri haline getirmektedir.

Detaylar için -> [Joomla](https://www.joomla.org/)

Bu kadar fazla tercih edilmesinden ötürü saldırganların önemli hedefi olmuş bir üründür. Joomla, öncesinde [improper access control](https://nvd.nist.gov/vuln/detail/CVE-2023-23752 "improper access control vulnerability (CVE-2023-23752)") güvenlik açığı ([CVE-2023-23752](https://nvd.nist.gov/vuln/detail/CVE-2023-23752)) aracılığıyla farklı kuruluşlara yönelik bir saldırıda hedef alındı.

Bu makalede, 22 Kasım 2023 tarihinde SonarCloud tarafından tespit edilen  ve PHP'deki bir hata nedeniyle ortaya çıkan [CVE-2024-21726](https://cve.mitre.org/cgi-bin/cvename.cgi?name=2024-21726)  XSS zafiyeti üzerine detaylı bir inceleme yapacağız. PHP'nin `mbstring` modülü (çok baytlı dize fonksiyonlarını ifade eder) içindeki bir tutarsızlık, saldırganların Joomla'da birden fazla XSS güvenlik açığını tetikleyen girdi sterilizasyonu sürecini atlamasına olanak tanıyabilir. Bu yazıda, bahsi geçen tutarsızlığın nasıl bir güvenlik zafiyetine yol açabileceğini ve saldırganların bu durumdan nasıl faydalanabileceğini açıklayacağız.

## Zafiyet

Joomla'yı tararken SonarCloud ilginç bir XSS sorunu bildirdi:

```php
<input type="hidden" name="forcedItemType" value="<?php echo $app->getInput()->get('forcedItemType', '', 'string'); ?>">

```
 <img src="../content-images/1.png" alt="1.Görsel">
(administrator/components/com_associations/tmpl/associations/modal.php)

Yönetici panelindeki bir ayarlar sayfasında bulunan kodun amacı:

`$app->getInput()->get()` metoduyla alınan `forcedItemType` adındaki bir kullanıcı girişi, bu gizli alanın değerini oluşturur. Ancak, burada bir güvenlik zafiyeti vardır. Kullanıcı girdisi doğrudan `echo` ile çıktıya dahil edilir ve bu, kötü niyetli bir kullanıcının XSS saldırısını gerçekleştirmesine olanak tanır.

Sorgu parametresini almak için kullanılan `get` yönteminin üçüncü argümanının `string` olarak ayarlandığına lütfen dikkat edin. Bu değer,  sorgu parametresine hangi filtrelerin uygulanması gerektiğini belirler.  Temel olarak, get yontemi, potansiyel olarak kotu amacli girisi temizlemek icin bir XSS sldirisini onlemesi gereken Joomla\\Filter\\InputFIlter sinifini kullanir.

Filtre mantığı oldukça karmaşıktır ve açıkça izin verilmeyen tüm HTML etiketlerini kaldırmak için cleanTags adlı bir yöntem kullanır. Sorgu parametreleri için hiçbir etikete izin verilmez.

Dolayısıyla, aşağıdaki örnek input için:

`herhangi-yazi<script>alert(10)</script>`

etiketleri kaldırılır, bu da şu çıktıyla sonuçlanır:

`herhangi-yazialert(10)`

Açılış etiketinden önceki karakterler (örneğin, yukarıdaki örnekte herhangi-yazi) `StringHelper::strpos` aracılığıyla açılış etiketinin ofseti (`$tagOpenStart`) belirlenerek ve ardından `StringHelper::substr` kullanılarak çıkarılır:

```php
// Bir etiket var mı? Eğer öyleyse kesinlikle bir '<' ile başlayacaktır.
$tagOpenStart = StringHelper::strpos($source, '<');
while ($tagOpenStart !== false) {
    // Get some information about the tag we are processing
    $preTag .= StringHelper::substr($postTag, 0, $tagOpenStart);
```

Örnek string `herhangi-yazi<script>alert(1)</script>` için, `StringHelper::substr` öğesine yapılan ilk çağrı `$preTag` değişkenine eklenen `herhangi-yazi` stringini döndürür:

 <img src="../content-images/2.png" alt="2.Görsel">

(Görseller https://www.sonarsource.com/blog/joomla-multiple-xss-vulnerabilities/ 'dan alınmıştır.)

### `mb_strpos` Fonksiyonu

`mb_strpos` fonksiyonu, bir string içinde belirli bir alt stringin ilk görüldüğü yerin index'ini döndürür. Bu fonksiyon, çoklu bayt karakter setlerinde doğru çalışmak üzere tasarlanmıştır. Ancak, geçersiz UTF-8 byte dizileri ile karşılaştığında beklenmedik davranışlar sergileyebilir.

Örnekle açıklayalım:

```php
// Geçersiz UTF-8 byte dizisi içeren bir string.
$string = "\xf0\x9fAAA<BB";

// '<' karakterinin pozisyonunu arıyoruz.
$position = mb_strpos($string, '<', 0, 'UTF-8');
echo "The position of '<' is: " . $position; // Çıktı: 4

```

Bu örnekte, `mb_strpos` fonksiyonu geçersiz byte dizisini (özellikle \\xf0\\x9f) bir karakter olarak değerlendirir ve `<` karakterinin pozisyonunu 4 olarak döndürür. Ancak, bu dizilim geçersiz olduğunda, PHP'nin bu byte'ları nasıl işleyeceği belirsiz olabilir ve bu, güvenlik açıklarına yol açabilir.

### `mb_substr` Fonksiyonu

`mb_substr` fonksiyonu, belirtilen string'in belirtilen başlangıç konumundan itibaren belirtilen uzunlukta bir alt string döndürür. Bu fonksiyon da çoklu bayt karakter setlerini destekler.

Örnekle açıklayalım:

```php
// Aynı geçersiz UTF-8 string
$string = "\xf0\x9fAAA<BB";

// mb_strpos'tan dönen index'i kullanarak alt stringi alıyoruz.
$result = mb_substr($string, 0, 4, 'UTF-8');
echo "The substring is: " . $result; // Çıktı: "\xf0\x9fAAA<B"

```

<img src="../content-images/3.png" alt="3.Görsel">

(Görseller https://www.sonarsource.com/blog/joomla-multiple-xss-vulnerabilities/ 'dan alınmıştır.)

Bu örnekte, `mb_substr` başlangıç byte'ına dayalı olarak takip eden byte'ları bir karakter olarak ele alır ve `<` karakteri dahil edilerek sonuç döndürülür. Bu, özellikle HTML içeriği kesme ve temizleme işlemlerinde sorunlara yol açabilir.

Bu PHP'nin eski sürümlerinde, mb_strpos ve mb_substr yöntemleri geçersiz UTF-8 dizilerini farklı şekilde ele alır.  Ne zaman mb_strpos UTF-8 öncü baytıyla karşılaştığında, bir sonraki devam baytını ayrıştırmaya çalışır ve bayt dizisinin tamamı okunana kadar baytlar. Eğer geçersiz bir bayt karşılaşıldığında, daha önce okunan tüm baytlar tek bir bayt olarak kabul edilir. Karakteri ve işleme geçersiz bayt ile yeniden başlar.

## Kaynakça:

https://www.sonarsource.com/blog/joomla-multiple-xss-vulnerabilities/

https://dayzerosec.com/vulns/2024/02/26/joomla-php-bug-introduces-multiple-xss-vulnerabilities.html

https://cve.mitre.org/cgi-bin/cvename.cgi?name=2024-21726

https://www.joomla.org/announcements/release-news/5904-joomla-5-0-3-and-4-4-3-security-and-bug-fix-release.html

https://developer.joomla.org/security-centre/929-20240205-core-inadequate-content-filtering-within-the-filter-code.html
