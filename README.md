
# ELASTİCSEARCH WATCHER 

![enter image description here](https://raw.githubusercontent.com/fatihaydnrepo/elasticsearch-watcher/main/watcher1.png)


Elasticsearch Watcher, Elasticsearch verilerinizdeki değişikliklere dayalı **uyarılar** ve **bildirimler** oluşturmanıza olanak tanıyan bir araçtır. Bu araç, Elasticsearch verilerindeki belirli olayları izler ve bu olayların meydana gelmesi durumunda belirli bir işlemi otomatik olarak başlatır. Örneğin, bir belirli bir sorgunun sonucunda belirli bir sayıda sonuç elde edildiğinde, belirli bir e-posta adresine bir e-posta gönderilmesi gibi. Elasticsearch Watcher, ölçeklenebilir bir uyarı sistemi sağlar ve genellikle büyük veri işleme ve izleme senaryolarında kullanılır.

Örneğin, log verilerindeki bir aramada son 5 dakikada çok sayıda 503 hatası olduğunu belirlediğinde, bir izleme (watch) yapılandırarak sistem yöneticisine e-posta göndermek için bir işlem gerçekleştirebilirsiniz.

### Nasıl çalışır ? 


Watcher ; bir trigger ,  bir input, bir işlem ve bir output'tan oluşur. Triggeri, watcher'ın ne zaman başlatılacağını belirler. İnput,  watcher payload'ı tanımlar ve veri kaynaklarını belirtir. İşlem, koşullar karşılandığında gerçekleştirilecek aksiyonları tanımlar. Output , işlem sonucunun nereye gönderileceğini belirtir.

Uygulamaya geçmeden önce, Watcher'ın nasıl çalıştığını anlamaya çalışalım.

-   Schedule
-   Query
-   Condition
-   Actions

**Schedule** :  Watcher 'ın ne zaman çalıştırılacağını ve belirli bir zaman dilimi boyunca hangi verilerin inceleneceğini belirlemek için kullanılır. 

**Query** :  Bir koşulla sonucu karşılaştırarak sorgulama yapmak için sorulacak bir sorudur ve DSL'deki tüm özellikler kullanılabilir.

**Condition** : Sorgumuzun sonucunu karşılaştırır ve eğer koşula uygunsa eylemi gerçekleştiririz.

**Actions** : Bilgi gönderme işlemini bu katman ile belirleyebiliriz. 

Örneğin, aşağıda paylaştığım kod bloğunu ele alabiliriz. 
```json
 PUT _watcher/watch/log_errors
{
  "metadata" : { 
    "color" : "red"
  },
  "trigger" : { 
    "schedule" : {
      "interval" : "5m"
    }
  },
  "input" : { 
    "search" : {
      "request" : {
        "indices" : "log-events",
        "body" : {
          "size" : 0,
          "query" : { "match" : { "status" : "error" } }
        }
      }
    }
  },
  "condition" : { 
    "compare" : { "ctx.payload.hits.total" : { "gt" : 5 }}
  },
  "transform" : { 
    "search" : {
        "request" : {
          "indices" : "log-events",
          "body" : {
            "query" : { "match" : { "status" : "error" } }
          }
        }
    }
  },
  "actions" : { 
    "my_webhook" : {
      "webhook" : {
        "method" : "POST",
        "host" : "mylisteninghost",
        "port" : 9200,
        "path" : "/{{watch_id}}",
        "body" : "Encountered {{ctx.payload.hits.total}} errors"
      }
    },
    "email_administrator" : {
      "email" : {
        "to" : "sys.admino@host.domain",
        "subject" : "Encountered {{ctx.payload.hits.total}} errors",
        "body" : "Too many error in the system, see attached data",
        "attachments" : {
          "attached_data" : {
            "data" : {
              "format" : "json"
            }
          }
        },
        "priority" : "high"
      }
    }
  }
}
```
Metadata - Watcher için isteğe bağlı olarak  statik meta verileri ekleyebilirsiniz.

Trigger - Zamanlamayı tetikler , izlemeyi her 5 dakikada bir çalıştırır.

Input - log-events ' de bulunan hataları arar ve yanıtı Watch payload'a aktarır.
  
Condition - Bu koşul, izleme sürecinde kullanılan verilerin değerlerini kontrol eder ve belirli bir eylem gerçekleştirmek için gerekli olan şartları sağlamak için tasarlanmıştır. 

Transform - Condition karşılanırsa, bu Transform tüm hataları watch payload'a  yükler.

Actions - email_administrator eylemi, sistem yöneticisine öncelikli bir e-posta gönderir. Hata içerenwatch payload e-postaya eklenir.

###  Watcher APIs

1. Watcher'ı oluşturmak için aşağıdaki kodu kullanabiliriz
```json
PUT _xpack/watcher/watch/watch_name  
{  
\\Body  
}
```
2. Get Watcher
> GET _xpack/watcher/watch/watch_name

3. Aktifleştirme/Pasifleştirme işlemi için ; 
> PUT _xpack/watcher/watch/watch_name/_activate/
>PUT _xpack/watcher/watch/watch_name/_deactivate

4. Execute Watcher
> PUT _xpack/watcher/watch/watch_name/_execute

Bu işlemleri yaptığımızda watcher'ın son hali aşağıdaki şekilde olmalı, ayrıca beat'lerdeki log kayıpları için, fatih@outlook.com adresine mail göndermesini de isteyelim, beat'in adıda deneme olsun.
```json
PUT _watcher/watch/deneme-watch
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [
          "beat-*"
        ],
        "body": {
          "query": {
            "match": {
              "message": "Failed to publish events"
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 0
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "profile": "standard",
        "to": "fatih@outlook.com",
        "subject": "Log kaybı tespit edildi",
        "body": {
          "text": "Deneme beat'lerinde log kaybı tespit edildi. Ayrıntılar aşağıdadır:\n\n{{#ctx.payload.hits.hits}}{{_source.message}}\n{{/ctx.payload.hits.hits}}"
        }
      }
    }
  }
}
```




  

