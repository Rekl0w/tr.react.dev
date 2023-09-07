---
title: 'Reacktif Efektlerin Yaşam Döngüsü'
---

<Intro>

Efektler bileşenlerden farklı bir yaşam döngüsü vardır. Bileşenler takılabilir, güncellenebilir veya çıkarılabilir.Efektler sadece iki şey yapabilir: bir şeyi senkronize etmeye başlamak için, ve daha sonra senkronizasyonu durdurmak için. Efektler zaman içinde değişen sahne ve durumlara bağlıysa bu döngü birden çok kez gerçekleşebilir. React, Efekt'inizin bağımlılıklarını doğru belirtip belirtmediğinizi kontrol etmek için bir linter kuralı sağlar. Bu, Efektinizin en son props ve state ile senkronize olmasını sağlar.

</Intro>

<YouWillLearn>

  Efektlerin yaşam döngüsü bir bileşenin yaşam döngüsünden nasıl farklıdır
- Her bir Efekt tek başına nasıl düşünülebilir
- Efekt ne zaman ve neden yeniden senkronize edilmesi gerektiği
- Efekt bağımlılıkları nasıl belirlenir?
- Bir değerin reaktif olması ne anlama gelir
- Boş bir bağımlılık dizisi ne anlama gelir?
- React, bir linter ile bağımlılıklarınızın doğru olduğunu nasıl doğrular
- Linter ile aynı fikirde olmadığınızda ne yapmalısınız

</YouWillLearn>

## Efektin Yaşam Döngüsü {/*the-lifecycle-of-an-effect*/}

Her React bileşeni aynı yaşam döngüsünden geçer:

- Bir bileşen ekrana eklendiğinde _monte_ edilir.
- Bir bileşen, genellikle bir etkileşime yanıt olarak yeni prop'lar veya state aldığında _updates_ yapar.
- Bir bileşen ekrandan kaldırıldığında _unmounts_ olur.

**Bileşenler hakkında düşünmek için iyi bir yol, ancak Efektler hakkında _değildir_.** Bunun yerine, her bir Efekt bileşeninizin yaşam döngüsünden bağımsız olarak düşünmeye çalışın. Bir Efekt [harici bir sistemin](/learn/synchronizing-with-effects) mevcut prop'lara ve state nasıl senkronize edileceğini açıklar. Kodunuz değiştikçe, senkronizasyonun daha sık veya daha seyrek yapılması gerekecektir.

Bu noktayı açıklamak için, bileşeninizi bir sohbet sunucusuna bağlayan bu Efekti düşünün:

```js
const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]);
  // ...
}
```

Efektinizin gövdesi **senkronizasyonun nasıl başlatılacağını belirtir:**

```js {2-3}
    // ...
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
    // ...
```

Efektiniz tarafından döndürülen temizleme işlevi **senkronizasyonun nasıl durdurulacağını belirtir:**

```js {5}
    // ...
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
    // ...
```

Sezgisel olarak, React'in bileşeniniz bağlandığında **senkronizasyonu başlatacağını** ve bileşeniniz ayrıldığında **senkronizasyonu durduracağını** düşünebilirsiniz. Ancak, bu hikayenin sonu değildir! Bazen, bileşen takılı kalırken **senkronizasyonu birden çok kez başlatmak ve durdurmak** da gerekebilir.

Şimdi bunun _neden_ gerekli olduğuna, _ne zaman_ gerçekleştiğine ve _bu davranışı _nasıl_ kontrol edebileceğinize bakalım.

<Note>

Bazı Efektler hiç temizleme fonksiyonu döndürmez. [Çoğu zaman,](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development) bir tane döndürmek isteyeceksiniz-ama döndürmezseniz, React boş bir temizleme fonksiyonu döndürmüşsünüz gibi davranacaktır.

</Note>

### Senkronizasyonun neden birden fazla kez yapılması gerekebilir {/*why-synchronization-may-need-to-happen-more-than-once*/}

Bu `ChatRoom` bileşeninin, kullanıcının bir açılır menüden seçtiği bir `roomId` prop'larını aldığını düşünün. Diyelim ki kullanıcı başlangıçta `roomId` olarak `"genel"` odasını seçti. Uygulamanız `"genel"` sohbet odasını görüntüler:

```js {3}
const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId /* "genel" */ }) {
  // ...
  return <h1>{roomId} odasına hoş geldiniz!</h1>;
}
```

UI görüntülendikten sonra, React **senkronizasyonu başlatmak için Efektinizi çalıştıracaktır.** `"genel"` odasına bağlanır:

```js {3,4}
function ChatRoom({ roomId /* "genel" */ }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // "genel" odaya bağlanır
    connection.connect();
    return () => {
      connection.disconnect(); // "genel" oda ile bağlantıyı keser
    };
  }, [roomId]);
  // ...
```

Buraya kadar her şey yolunda.

Daha sonra, kullanıcı açılır menüden farklı bir oda seçer (örneğin, `"seyahat"`). İlk olarak, React kullanıcı arayüzünü güncelleyecektir:

```js {1}
function ChatRoom({ roomId /* "seyahat" */ }) {
  // ...
  return <h1>{roomId} odasına hoş geldiniz!</h1>;
}
```

Bundan sonra ne olması gerektiğini düşünün. Kullanıcı, kullanıcı arayüzünde seçili sohbet odasının `"seyahat"` olduğunu görür. Ancak, son kez çalışan Efekt hala `"genel"` odasına bağlı. **`roomId` prop'u değişti, bu nedenle Efektinizin o zaman yaptığı şey (`"genel"` odasına bağlanmak) artık kullanıcı arayüzüyle eşleşmiyor.**

Bu noktada, React'in iki şey yapmasını istersiniz:

1. Eski `roomId` ile senkronizasyonu durdurun (`"genel"` oda ile bağlantıyı kesin)
2. Yeni `roomId` ile senkronizasyonu başlatın (`"seyahat"` odasına bağlanın)

**Neyse ki, React'e bunların her ikisini de nasıl yapacağını zaten öğrettiniz.** Efektinizin gövdesi senkronizasyonun nasıl başlatılacağını ve temizleme fonksiyonunuz da senkronizasyonun nasıl durdurulacağını belirtir. React'in şimdi yapması gereken tek şey, bunları doğru sırada ve doğru prop ve state ile çağırmaktır. Bunun tam olarak nasıl gerçekleştiğini görelim.

### React Etkinizi Nasıl Yeniden Senkronize Eder? {/*how-react-re-synchronizes-your-effect*/}

Hatırlayın, `ChatRoom` bileşeniniz `roomId` özelliği için yeni bir değer aldı. Eskiden `"general"` idi ve şimdi `"travel"` oldu. React'in sizi farklı bir odaya yeniden bağlamak için Efektinizi yeniden senkronize etmesi gerekiyor.

React, **senkronizasyonu durdurmak için,** Efektinizin `"general"` odasına bağlandıktan sonra döndürdüğü temizleme fonksiyonunu çağıracaktır. roomId` `"general"` olduğu için, temizleme fonksiyonu `"general"` odasıyla bağlantıyı keser:

```js {6}
function ChatRoom({ roomId /* "genel" */ }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // "genel" odaya bağlanır
    connection.connect();
    return () => {
      connection.disconnect(); // "genel" oda ile bağlantıyı keser
    };
    // ...
```

Ardından React, bu render sırasında sağladığınız Efekti çalıştıracaktır. Bu sefer, `roomId` `"seyahat"` olduğundan, `"seyahat"` sohbet odasına **senkronize olmaya** başlayacaktır (sonunda temizleme fonksiyonu da çağrılana kadar):

```js {3,4}
function ChatRoom({ roomId /* "seyahat" */ }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // "seyahat" odasına bağlanır
    connection.connect();
    // ...
```

Bu sayede, artık kullanıcının kullanıcı arayüzünde seçtiği odaya bağlanmış olursunuz. Felaket önlendi!

Bileşeniniz farklı bir `roomId` ile yeniden oluşturulduktan sonra her seferinde Efektiniz yeniden senkronize olacaktır. Örneğin, kullanıcı `roomId`yi `"seyahat"`ten `"müzik"`e değiştirdi diyelim. React, temizleme fonksiyonunu çağırarak (sizi `"seyahat"` odasından ayırarak) Efektinizin senkronizasyonunu tekrar **durdurur**. Ardından, gövdesini yeni `roomId` prop ile çalıştırarak (sizi `"müzik"` odasına bağlayarak) tekrar **senkronize etmeye** başlayacaktır.

Son olarak, kullanıcı farklı bir ekrana geçtiğinde, `ChatRoom` bağlantıyı kaldırır. Artık bağlı kalmaya hiç gerek yok. React, Efektinizi son bir kez **senkronize etmeyi durdurur** ve sizi `"müzik"` sohbet odasından ayırır.

### Efektin bakış açısından düşünmek {/*thinking-from-the-effects-perspective*/}

Şimdi `ChatRoom' bileşeninin bakış açısından olan her şeyi özetleyelim:

1. `ChatRoom` `roomId` `"genel"` olarak ayarlanmış şekilde monte edildi
1. `ChatRoom`, `roomId` değeri `"seyahat"` olarak ayarlanarak güncellendi
1. `ChatRoom`, `roomId` değeri `"müzik"` olarak ayarlanarak güncellendi
1. ChatRoom` bağlanmamış

Bileşenin yaşam döngüsündeki bu noktaların her biri sırasında, Efektiniz farklı şeyler yaptı:

1. Efektiniz `"genel"` odaya bağlandı
1. Efektinizin `"genel"` oda ile bağlantısı kesildi ve `"seyahat"` odasına bağlandı
1. Efektinizin "seyahat" odasıyla bağlantısı kesildi ve "müzik" odasına bağlandı
1. Efektinizin "müzik" odasıyla bağlantısı kesildi

Şimdi olanları bir de Efektin kendi perspektifinden düşünelim:

```js
  useEffect(() => {
    // Efektiniz roomId ile belirtilen odaya bağlandı...
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      // ...bağlantısı kesilene kadar
      connection.disconnect();
    };
  }, [roomId]);
```

Bu kodun yapısı, olanları birbiriyle örtüşmeyen bir dizi zaman dilimi olarak görmeniz için size ilham verebilir:

1. Efektiniz `"genel"` odaya bağlandı (bağlantısı kesilene kadar)
1. Efektiniz `"seyahat"` odasına bağlı (bağlantısı kesilene kadar)
1. Efektiniz `"müzik"` odasına bağlı (bağlantısı kesilene kadar)

Önceden, bileşenin bakış açısından düşünüyordunuz. Bileşenin perspektifinden baktığınızda, Efektleri "render işleminden sonra" veya "unmount işleminden önce" gibi belirli bir zamanda ateşlenen "geri aramalar" veya "yaşam döngüsü olayları" olarak düşünmek cazip geliyordu. Bu düşünce tarzı çok hızlı bir şekilde karmaşıklaşır, bu nedenle kaçınmak en iyisidir.

**Bunun yerine, her zaman bir seferde tek bir başlatma/durdurma döngüsüne odaklanın. Bir bileşenin takılıyor, güncelleniyor ya da sökülüyor olması önemli olmamalıdır. Yapmanız gereken tek şey senkronizasyonun nasıl başlatılacağını ve nasıl durdurulacağını açıklamaktır. Bunu iyi yaparsanız, Efektiniz ihtiyaç duyulduğu kadar çok kez başlatılmaya ve durdurulmaya dayanıklı olacaktır.**

Bu size, JSX oluşturan işleme mantığını yazarken bir bileşenin monte edilip edilmediğini veya güncellenip güncellenmediğini nasıl düşünmediğinizi hatırlatabilir. Siz ekranda ne olması gerektiğini tanımlarsınız ve React [gerisini çözer](/learn/reacting-to-input-with-state)

### React, Efektinizin yeniden senkronize olabileceğini nasıl doğrular {/*how-react-verifies-that-your-effect-can-re-synchronize*/}

İşte oynayabileceğiniz canlı bir örnek. ChatRoom bileşenini bağlamak için "Sohbeti aç" düğmesine basın:

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);
  return <h1>{roomId} odasına hoş geldiniz!</h1>;
}

export default function App() {
  const [roomId, setRoomId] = useState('genel');
  const [show, setShow] = useState(false);
  return (
    <>
      <label>
        Sohbet odasını seçin:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="genel">genel</option>
          <option value="seyahat">seyahat</option>
          <option value="müzik">müzik</option>
        </select>
      </label>
      <button onClick={() => setShow(!show)}>
        {show ? 'Sohbeti kapat' : 'Sohbeti aç'}
      </button>
      {show && <hr />}
      {show && <ChatRoom roomId={roomId} />}
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Bağlanmak "' + roomId + '" oda ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Bağlantısı kesildi "' + roomId + '" oda ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

Bileşen ilk kez bağlandığında üç günlük gördüğünüze dikkat edin:

1. `✅ "Genel" odaya bağlanma https://localhost:1234...` *(sadece geliştirme)*
1. `❌ "Genel" oda ile bağlantı kesildi https://localhost:1234.` *(sadece geliştirme)*
1. `✅ Adresinden "genel" odasına bağlanıyor https://localhost:1234...`

İlk iki günlük yalnızca geliştirmeye yöneliktir. Geliştirme aşamasında, React her bileşeni her zaman bir kez yeniden bağlar.

**React, Efektinizin yeniden senkronize olup olamayacağını, onu geliştirme aşamasında bunu hemen yapmaya zorlayarak doğrular.** Bu size kapı kilidinin çalışıp çalışmadığını kontrol etmek için bir kapıyı açıp fazladan bir kez kapatmayı hatırlatabilir. React, kontrol etmek için geliştirme sırasında Efektinizi fazladan bir kez başlatır ve durdurur [temizlemeyi iyi uyguladığınızı](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development)

Efektinizin pratikte yeniden senkronize olmasının ana nedeni, kullandığı bazı verilerin değişmiş olmasıdır. Yukarıdaki sanal alanda, seçili sohbet odasını değiştirin. RoomId` değiştiğinde Efektinizin nasıl yeniden senkronize olduğuna dikkat edin.

Ancak, yeniden senkronizasyonun gerekli olduğu daha sıra dışı durumlar da vardır. Örneğin, sohbet açıkken yukarıdaki sanal alanda `sunucuUrl`yi düzenlemeyi deneyin. Kodda yaptığınız düzenlemelere yanıt olarak Efekt'in nasıl yeniden senkronize olduğuna dikkat edin. Gelecekte React, yeniden senkronizasyona dayanan daha fazla özellik ekleyebilir.

### React, Efekti yeniden senkronize etmesi gerektiğini nasıl anlar {/*how-react-knows-that-it-needs-to-re-synchronize-the-effect*/}

React'in `roomId` değiştikten sonra Efektinizin yeniden senkronize edilmesi gerektiğini nasıl bildiğini merak ediyor olabilirsiniz. Çünkü *React'e* kodunun `roomId`'ye bağlı olduğunu [bağımlılıklar listesi:](/learn/synchronizing-with-effects#step-2-specify-the-effect-dependencies) içine dahil ederek söylediniz.


```js {1,3,8}
function ChatRoom({ roomId }) { // roomId özelliği zaman içinde değişebilir
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Bu Efektte roomId'yi okur
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]); // Böylece React'e bu Efektin roomId'ye "bağlı" olduğunu söylersiniz
  // ...
```

Here's how this works:

1. You knew `roomId` is a prop, which means it can change over time.
2. You knew that your Effect reads `roomId` (so its logic depends on a value that may change later).
3. This is why you specified it as your Effect's dependency (so that it re-synchronizes when `roomId` changes).

Bileşeniniz yeniden oluşturulduktan sonra React her seferinde geçtiğiniz bağımlılıklar dizisine bakacaktır. Dizideki değerlerden herhangi biri, önceki render sırasında geçtiğiniz aynı noktadaki değerden farklıysa, React Efektinizi yeniden senkronize edecektir.

Örneğin, ilk render sırasında `["genel"]` değerini geçtiyseniz ve daha sonra bir sonraki render sırasında `["seyahat"]` değerini geçtiyseniz, React `"genel"` ve `"seyahat"` değerlerini karşılaştıracaktır. Bunlar farklı değerlerdir ([`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is) ile karşılaştırıldığında), bu nedenle React Efektinizi yeniden senkronize edecektir. Öte yandan, bileşeniniz yeniden render edilirse ancak `roomId` değişmediyse, Efektiniz aynı odaya bağlı kalacaktır.

### Her Efekt ayrı bir senkronizasyon sürecini temsil eder {/*each-effect-represents-a-separate-synchronization-process*/}

Yalnızca bu mantığın daha önce yazdığınız bir Efekt ile aynı anda çalışması gerektiği için Efektinize ilgisiz bir mantık eklemekten kaçının. Örneğin, kullanıcı odayı ziyaret ettiğinde bir analiz olayı göndermek istediğinizi varsayalım. Zaten `roomId`ye bağlı bir Efektiniz var, bu nedenle analitik çağrısını oraya eklemek isteyebilirsiniz:

```js {3}
function ChatRoom({ roomId }) {
  useEffect(() => {
    logVisit(roomId);
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]);
  // ...
}
```

Ancak daha sonra bu Efekte bağlantıyı yeniden kurması gereken başka bir bağımlılık eklediğinizi düşünün. Bu Etki yeniden senkronize olursa, aynı oda için `logVisit(roomId)` çağrısı da yapacaktır, ki bunu istememiştiniz. Ziyaretin günlüğe kaydedilmesi **bağlantıdan ayrı bir süreçtir**. Bunları iki ayrı Efekt olarak yazın:

```js {2-4}
function ChatRoom({ roomId }) {
  useEffect(() => {
    logVisit(roomId);
  }, [roomId]);

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    // ...
  }, [roomId]);
  // ...
}
```

**Kodunuzdaki her bir Efekt ayrı ve bağımsız bir senkronizasyon sürecini temsil etmelidir.**

Yukarıdaki örnekte, bir Efektin silinmesi diğer Efektin mantığını bozmayacaktır. Bu, farklı şeyleri senkronize ettiklerinin iyi bir göstergesidir ve bu nedenle onları ayırmak mantıklıdır. Öte yandan, uyumlu bir mantık parçasını ayrı Efektlere bölerseniz, kod "daha temiz" görünebilir ancak [bakımı daha zor](/learn/you-might-not-need-an-effect#chains-of-computations) olacaktır. Bu nedenle, kodun daha temiz görünüp görünmediğini değil, süreçlerin aynı mı yoksa ayrı mı olduğunu düşünmelisiniz.

## Efektler reaktif değerlere "tepki verir" {/*effects-react-to-reactive-values*/}

Efektiniz iki değişkeni (`serverUrl` ve `roomId`) okuyor, ancak bağımlılık olarak yalnızca `roomId` belirtmişsiniz:

```js {5,10}
const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]);
  // ...
}
```

Neden `serverUrl` bir bağımlılık olmak zorunda değil?

Çünkü `serverUrl` yeniden oluşturma nedeniyle asla değişmez. Bileşen kaç kez yeniden oluşturulursa oluşturulsun ve nedeni ne olursa olsun her zaman aynıdır. SunucuUrl` asla değişmediğinden, bunu bir bağımlılık olarak belirtmek mantıklı olmaz. Sonuçta, bağımlılıklar yalnızca zaman içinde değiştiklerinde bir şey yaparlar!

Öte yandan, `roomId` yeniden oluşturmada farklı olabilir. **Bileşen içinde bildirilen prop'lar, state ve diğer değerler _reaktiftir_ çünkü render sırasında hesaplanırlar ve React veri akışına katılırlar.**

Eğer `serverUrl` bir state değişkeni olsaydı, reaktif olurdu. Reaktif değerler bağımlılıklara dahil edilmelidir:

```js {2,5,10}
function ChatRoom({ roomId }) { // Prop'ların zaman içinde değişimi
  const [serverUrl, setServerUrl] = useState('https://localhost:1234'); // State zaman içinde değişebilir

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Etkiniz prop'ları ve state okur
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId, serverUrl]); // Böylece React'e bu Efektin proplara ve state "bağlı" olduğunu söylersiniz
  // ...
}
```

Sunucu URL`sini bir bağımlılık olarak dahil ederek, Efektin değiştikten sonra yeniden senkronize olmasını sağlarsınız.

Seçili sohbet odasını değiştirmeyi deneyin veya bu sanal alanda sunucu URL'sini düzenleyin:

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

function ChatRoom({ roomId }) {
  const [serverUrl, setServerUrl] = useState('https://localhost:1234');

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, serverUrl]);

  return (
    <>
      <label>
        Server URL:{' '}
        <input
          value={serverUrl}
          onChange={e => setServerUrl(e.target.value)}
        />
      </label>
      <h1>{roomId} odasına hoş geldiniz!</h1>
    </>
  );
}

export default function App() {
  const [roomId, setRoomId] = useState('genel');
  return (
    <>
      <label>
        Sohbet odasını seçin:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="genel">general</option>
          <option value="seyahat">seyahat</option>
          <option value="müzik">müzik</option>
        </select>
      </label>
      <hr />
      <ChatRoom roomId={roomId} />
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // Gerçek bir uygulama aslında sunucuya bağlanır
  return {
    connect() {
      console.log('✅ Bağlanmak "' + roomId + '" oda ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Bağlantısı kesildi "' + roomId + '" oda ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

Whenever you change a reactive value like `roomId` or `serverUrl`, the Effect re-connects to the chat server.

### What an Effect with empty dependencies means {/*what-an-effect-with-empty-dependencies-means*/}

What happens if you move both `serverUrl` and `roomId` outside the component?

```js {1,2}
const serverUrl = 'https://localhost:1234';
const roomId = 'general';

function ChatRoom() {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // ✅ All dependencies declared
  // ...
}
```

Now your Effect's code does not use *any* reactive values, so its dependencies can be empty (`[]`).

Thinking from the component's perspective, the empty `[]` dependency array means this Effect connects to the chat room only when the component mounts, and disconnects only when the component unmounts. (Keep in mind that React would still [re-synchronize it an extra time](#how-react-verifies-that-your-effect-can-re-synchronize) in development to stress-test your logic.)


<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

const serverUrl = 'https://localhost:1234';
const roomId = 'general';

function ChatRoom() {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, []);
  return <h1>Welcome to the {roomId} room!</h1>;
}

export default function App() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Close chat' : 'Open chat'}
      </button>
      {show && <hr />}
      {show && <ChatRoom />}
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '" room at ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

However, if you [think from the Effect's perspective,](#thinking-from-the-effects-perspective) you don't need to think about mounting and unmounting at all. What's important is you've specified what your Effect does to start and stop synchronizing. Today, it has no reactive dependencies. But if you ever want the user to change `roomId` or `serverUrl` over time (and they would become reactive), your Effect's code won't change. You will only need to add them to the dependencies.

### All variables declared in the component body are reactive {/*all-variables-declared-in-the-component-body-are-reactive*/}

Props and state aren't the only reactive values. Values that you calculate from them are also reactive. If the props or state change, your component will re-render, and the values calculated from them will also change. This is why all variables from the component body used by the Effect should be in the Effect dependency list.

Let's say that the user can pick a chat server in the dropdown, but they can also configure a default server in settings. Suppose you've already put the settings state in a [context](/learn/scaling-up-with-reducer-and-context) so you read the `settings` from that context. Now you calculate the `serverUrl` based on the selected server from props and the default server:

```js {3,5,10}
function ChatRoom({ roomId, selectedServerUrl }) { // roomId is reactive
  const settings = useContext(SettingsContext); // settings is reactive
  const serverUrl = selectedServerUrl ?? settings.defaultServerUrl; // serverUrl is reactive
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Your Effect reads roomId and serverUrl
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId, serverUrl]); // So it needs to re-synchronize when either of them changes!
  // ...
}
```

In this example, `serverUrl` is not a prop or a state variable. It's a regular variable that you calculate during rendering. But it's calculated during rendering, so it can change due to a re-render. This is why it's reactive.

**All values inside the component (including props, state, and variables in your component's body) are reactive. Any reactive value can change on a re-render, so you need to include reactive values as Effect's dependencies.**

In other words, Effects "react" to all values from the component body.

<DeepDive>

#### Can global or mutable values be dependencies? {/*can-global-or-mutable-values-be-dependencies*/}

Mutable values (including global variables) aren't reactive.

**A mutable value like [`location.pathname`](https://developer.mozilla.org/en-US/docs/Web/API/Location/pathname) can't be a dependency.** It's mutable, so it can change at any time completely outside of the React rendering data flow. Changing it wouldn't trigger a re-render of your component. Therefore, even if you specified it in the dependencies, React *wouldn't know* to re-synchronize the Effect when it changes. This also breaks the rules of React because reading mutable data during rendering (which is when you calculate the dependencies) breaks [purity of rendering.](/learn/keeping-components-pure) Instead, you should read and subscribe to an external mutable value with [`useSyncExternalStore`.](/learn/you-might-not-need-an-effect#subscribing-to-an-external-store)

**A mutable value like [`ref.current`](/reference/react/useRef#reference) or things you read from it also can't be a dependency.** The ref object returned by `useRef` itself can be a dependency, but its `current` property is intentionally mutable. It lets you [keep track of something without triggering a re-render.](/learn/referencing-values-with-refs) But since changing it doesn't trigger a re-render, it's not a reactive value, and React won't know to re-run your Effect when it changes.

As you'll learn below on this page, a linter will check for these issues automatically.

</DeepDive>

### React verifies that you specified every reactive value as a dependency {/*react-verifies-that-you-specified-every-reactive-value-as-a-dependency*/}

If your linter is [configured for React,](/learn/editor-setup#linting) it will check that every reactive value used by your Effect's code is declared as its dependency. For example, this is a lint error because both `roomId` and `serverUrl` are reactive:

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

function ChatRoom({ roomId }) { // roomId is reactive
  const [serverUrl, setServerUrl] = useState('https://localhost:1234'); // serverUrl is reactive

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, []); // <-- Something's wrong here!

  return (
    <>
      <label>
        Server URL:{' '}
        <input
          value={serverUrl}
          onChange={e => setServerUrl(e.target.value)}
        />
      </label>
      <h1>Welcome to the {roomId} room!</h1>
    </>
  );
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <hr />
      <ChatRoom roomId={roomId} />
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '" room at ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

This may look like a React error, but really React is pointing out a bug in your code. Both `roomId` and `serverUrl` may change over time, but you're forgetting to re-synchronize your Effect when they change. You will remain connected to the initial `roomId` and `serverUrl` even after the user picks different values in the UI.

To fix the bug, follow the linter's suggestion to specify `roomId` and `serverUrl` as dependencies of your Effect:

```js {9}
function ChatRoom({ roomId }) { // roomId is reactive
  const [serverUrl, setServerUrl] = useState('https://localhost:1234'); // serverUrl is reactive
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [serverUrl, roomId]); // ✅ All dependencies declared
  // ...
}
```

Try this fix in the sandbox above. Verify that the linter error is gone, and the chat re-connects when needed.

<Note>

In some cases, React *knows* that a value never changes even though it's declared inside the component. For example, the [`set` function](/reference/react/useState#setstate) returned from `useState` and the ref object returned by [`useRef`](/reference/react/useRef) are *stable*--they are guaranteed to not change on a re-render. Stable values aren't reactive, so you may omit them from the list. Including them is allowed: they won't change, so it doesn't matter.

</Note>

### What to do when you don't want to re-synchronize {/*what-to-do-when-you-dont-want-to-re-synchronize*/}

In the previous example, you've fixed the lint error by listing `roomId` and `serverUrl` as dependencies.

**However, you could instead "prove" to the linter that these values aren't reactive values,** i.e. that they *can't* change as a result of a re-render. For example, if `serverUrl` and `roomId` don't depend on rendering and always have the same values, you can move them outside the component. Now they don't need to be dependencies:

```js {1,2,11}
const serverUrl = 'https://localhost:1234'; // serverUrl is not reactive
const roomId = 'general'; // roomId is not reactive

function ChatRoom() {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // ✅ All dependencies declared
  // ...
}
```

You can also move them *inside the Effect.* They aren't calculated during rendering, so they're not reactive:

```js {3,4,10}
function ChatRoom() {
  useEffect(() => {
    const serverUrl = 'https://localhost:1234'; // serverUrl is not reactive
    const roomId = 'general'; // roomId is not reactive
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // ✅ All dependencies declared
  // ...
}
```

**Effects are reactive blocks of code.** They re-synchronize when the values you read inside of them change. Unlike event handlers, which only run once per interaction, Effects run whenever synchronization is necessary.

**You can't "choose" your dependencies.** Your dependencies must include every [reactive value](#all-variables-declared-in-the-component-body-are-reactive) you read in the Effect. The linter enforces this. Sometimes this may lead to problems like infinite loops and to your Effect re-synchronizing too often. Don't fix these problems by suppressing the linter! Here's what to try instead:

* **Check that your Effect represents an independent synchronization process.** If your Effect doesn't synchronize anything, [it might be unnecessary.](/learn/you-might-not-need-an-effect) If it synchronizes several independent things, [split it up.](#each-effect-represents-a-separate-synchronization-process)

* **If you want to read the latest value of props or state without "reacting" to it and re-synchronizing the Effect,** you can split your Effect into a reactive part (which you'll keep in the Effect) and a non-reactive part (which you'll extract into something called an _Effect Event_). [Read about separating Events from Effects.](/learn/separating-events-from-effects)

* **Avoid relying on objects and functions as dependencies.** If you create objects and functions during rendering and then read them from an Effect, they will be different on every render. This will cause your Effect to re-synchronize every time. [Read more about removing unnecessary dependencies from Effects.](/learn/removing-effect-dependencies)

<Pitfall>

The linter is your friend, but its powers are limited. The linter only knows when the dependencies are *wrong*. It doesn't know *the best* way to solve each case. If the linter suggests a dependency, but adding it causes a loop, it doesn't mean the linter should be ignored. You need to change the code inside (or outside) the Effect so that that value isn't reactive and doesn't *need* to be a dependency.

If you have an existing codebase, you might have some Effects that suppress the linter like this:

```js {3-4}
useEffect(() => {
  // ...
  // 🔴 Avoid suppressing the linter like this:
  // eslint-ignore-next-line react-hooks/exhaustive-deps
}, []);
```

On the [next](/learn/separating-events-from-effects) [pages](/learn/removing-effect-dependencies), you'll learn how to fix this code without breaking the rules. It's always worth fixing!

</Pitfall>

<Recap>

- Components can mount, update, and unmount.
- Each Effect has a separate lifecycle from the surrounding component.
- Each Effect describes a separate synchronization process that can *start* and *stop*.
- When you write and read Effects, think from each individual Effect's perspective (how to start and stop synchronization) rather than from the component's perspective (how it mounts, updates, or unmounts).
- Values declared inside the component body are "reactive".
- Reactive values should re-synchronize the Effect because they can change over time.
- The linter verifies that all reactive values used inside the Effect are specified as dependencies.
- All errors flagged by the linter are legitimate. There's always a way to fix the code to not break the rules.

</Recap>

<Challenges>

#### Fix reconnecting on every keystroke {/*fix-reconnecting-on-every-keystroke*/}

In this example, the `ChatRoom` component connects to the chat room when the component mounts, disconnects when it unmounts, and reconnects when you select a different chat room. This behavior is correct, so you need to keep it working.

However, there is a problem. Whenever you type into the message box input at the bottom, `ChatRoom` *also* reconnects to the chat. (You can notice this by clearing the console and typing into the input.) Fix the issue so that this doesn't happen.

<Hint>

You might need to add a dependency array for this Effect. What dependencies should be there?

</Hint>

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  const [message, setMessage] = useState('');

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  });

  return (
    <>
      <h1>Welcome to the {roomId} room!</h1>
      <input
        value={message}
        onChange={e => setMessage(e.target.value)}
      />
    </>
  );
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <hr />
      <ChatRoom roomId={roomId} />
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '" room at ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

<Solution>

This Effect didn't have a dependency array at all, so it re-synchronized after every re-render. First, add a dependency array. Then, make sure that every reactive value used by the Effect is specified in the array. For example, `roomId` is reactive (because it's a prop), so it should be included in the array. This ensures that when the user selects a different room, the chat reconnects. On the other hand, `serverUrl` is defined outside the component. This is why it doesn't need to be in the array.

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

const serverUrl = 'https://localhost:1234';

function ChatRoom({ roomId }) {
  const [message, setMessage] = useState('');

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  return (
    <>
      <h1>Welcome to the {roomId} room!</h1>
      <input
        value={message}
        onChange={e => setMessage(e.target.value)}
      />
    </>
  );
}

export default function App() {
  const [roomId, setRoomId] = useState('general');
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <hr />
      <ChatRoom roomId={roomId} />
    </>
  );
}
```

```js chat.js
export function createConnection(serverUrl, roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '" room at ' + serverUrl + '...');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
    }
  };
}
```

```css
input { display: block; margin-bottom: 20px; }
button { margin-left: 10px; }
```

</Sandpack>

</Solution>

#### Switch synchronization on and off {/*switch-synchronization-on-and-off*/}

In this example, an Effect subscribes to the window [`pointermove`](https://developer.mozilla.org/en-US/docs/Web/API/Element/pointermove_event) event to move a pink dot on the screen. Try hovering over the preview area (or touching the screen if you're on a mobile device), and see how the pink dot follows your movement.

There is also a checkbox. Ticking the checkbox toggles the `canMove` state variable, but this state variable is not used anywhere in the code. Your task is to change the code so that when `canMove` is `false` (the checkbox is ticked off), the dot should stop moving. After you toggle the checkbox back on (and set `canMove` to `true`), the box should follow the movement again. In other words, whether the dot can move or not should stay synchronized to whether the checkbox is checked.

<Hint>

You can't declare an Effect conditionally. However, the code inside the Effect can use conditions!

</Hint>

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function App() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [canMove, setCanMove] = useState(true);

  useEffect(() => {
    function handleMove(e) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
    window.addEventListener('pointermove', handleMove);
    return () => window.removeEventListener('pointermove', handleMove);
  }, []);

  return (
    <>
      <label>
        <input type="checkbox"
          checked={canMove}
          onChange={e => setCanMove(e.target.checked)} 
        />
        The dot is allowed to move
      </label>
      <hr />
      <div style={{
        position: 'absolute',
        backgroundColor: 'pink',
        borderRadius: '50%',
        opacity: 0.6,
        transform: `translate(${position.x}px, ${position.y}px)`,
        pointerEvents: 'none',
        left: -20,
        top: -20,
        width: 40,
        height: 40,
      }} />
    </>
  );
}
```

```css
body {
  height: 200px;
}
```

</Sandpack>

<Solution>

One solution is to wrap the `setPosition` call into an `if (canMove) { ... }` condition:

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function App() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [canMove, setCanMove] = useState(true);

  useEffect(() => {
    function handleMove(e) {
      if (canMove) {
        setPosition({ x: e.clientX, y: e.clientY });
      }
    }
    window.addEventListener('pointermove', handleMove);
    return () => window.removeEventListener('pointermove', handleMove);
  }, [canMove]);

  return (
    <>
      <label>
        <input type="checkbox"
          checked={canMove}
          onChange={e => setCanMove(e.target.checked)} 
        />
        The dot is allowed to move
      </label>
      <hr />
      <div style={{
        position: 'absolute',
        backgroundColor: 'pink',
        borderRadius: '50%',
        opacity: 0.6,
        transform: `translate(${position.x}px, ${position.y}px)`,
        pointerEvents: 'none',
        left: -20,
        top: -20,
        width: 40,
        height: 40,
      }} />
    </>
  );
}
```

```css
body {
  height: 200px;
}
```

</Sandpack>

Alternatively, you could wrap the *event subscription* logic into an `if (canMove) { ... }` condition:

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function App() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [canMove, setCanMove] = useState(true);

  useEffect(() => {
    function handleMove(e) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
    if (canMove) {
      window.addEventListener('pointermove', handleMove);
      return () => window.removeEventListener('pointermove', handleMove);
    }
  }, [canMove]);

  return (
    <>
      <label>
        <input type="checkbox"
          checked={canMove}
          onChange={e => setCanMove(e.target.checked)} 
        />
        The dot is allowed to move
      </label>
      <hr />
      <div style={{
        position: 'absolute',
        backgroundColor: 'pink',
        borderRadius: '50%',
        opacity: 0.6,
        transform: `translate(${position.x}px, ${position.y}px)`,
        pointerEvents: 'none',
        left: -20,
        top: -20,
        width: 40,
        height: 40,
      }} />
    </>
  );
}
```

```css
body {
  height: 200px;
}
```

</Sandpack>

In both of these cases, `canMove` is a reactive variable that you read inside the Effect. This is why it must be specified in the list of Effect dependencies. This ensures that the Effect re-synchronizes after every change to its value.

</Solution>

#### Investigate a stale value bug {/*investigate-a-stale-value-bug*/}

In this example, the pink dot should move when the checkbox is on, and should stop moving when the checkbox is off. The logic for this has already been implemented: the `handleMove` event handler checks the `canMove` state variable.

However, for some reason, the `canMove` state variable inside `handleMove` appears to be "stale": it's always `true`, even after you tick off the checkbox. How is this possible? Find the mistake in the code and fix it.

<Hint>

If you see a linter rule being suppressed, remove the suppression! That's where the mistakes usually are.

</Hint>

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function App() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [canMove, setCanMove] = useState(true);

  function handleMove(e) {
    if (canMove) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
  }

  useEffect(() => {
    window.addEventListener('pointermove', handleMove);
    return () => window.removeEventListener('pointermove', handleMove);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <>
      <label>
        <input type="checkbox"
          checked={canMove}
          onChange={e => setCanMove(e.target.checked)} 
        />
        The dot is allowed to move
      </label>
      <hr />
      <div style={{
        position: 'absolute',
        backgroundColor: 'pink',
        borderRadius: '50%',
        opacity: 0.6,
        transform: `translate(${position.x}px, ${position.y}px)`,
        pointerEvents: 'none',
        left: -20,
        top: -20,
        width: 40,
        height: 40,
      }} />
    </>
  );
}
```

```css
body {
  height: 200px;
}
```

</Sandpack>

<Solution>

The problem with the original code was suppressing the dependency linter. If you remove the suppression, you'll see that this Effect depends on the `handleMove` function. This makes sense: `handleMove` is declared inside the component body, which makes it a reactive value. Every reactive value must be specified as a dependency, or it can potentially get stale over time!

The author of the original code has "lied" to React by saying that the Effect does not depend (`[]`) on any reactive values. This is why React did not re-synchronize the Effect after `canMove` has changed (and `handleMove` with it). Because React did not re-synchronize the Effect, the `handleMove` attached as a listener is the `handleMove` function created during the initial render. During the initial render, `canMove` was `true`, which is why `handleMove` from the initial render will forever see that value.

**If you never suppress the linter, you will never see problems with stale values.** There are a few different ways to solve this bug, but you should always start by removing the linter suppression. Then change the code to fix the lint error.

You can change the Effect dependencies to `[handleMove]`, but since it's going to be a newly defined function for every render, you might as well remove dependencies array altogether. Then the Effect *will* re-synchronize after every re-render:

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function App() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [canMove, setCanMove] = useState(true);

  function handleMove(e) {
    if (canMove) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
  }

  useEffect(() => {
    window.addEventListener('pointermove', handleMove);
    return () => window.removeEventListener('pointermove', handleMove);
  });

  return (
    <>
      <label>
        <input type="checkbox"
          checked={canMove}
          onChange={e => setCanMove(e.target.checked)} 
        />
        The dot is allowed to move
      </label>
      <hr />
      <div style={{
        position: 'absolute',
        backgroundColor: 'pink',
        borderRadius: '50%',
        opacity: 0.6,
        transform: `translate(${position.x}px, ${position.y}px)`,
        pointerEvents: 'none',
        left: -20,
        top: -20,
        width: 40,
        height: 40,
      }} />
    </>
  );
}
```

```css
body {
  height: 200px;
}
```

</Sandpack>

This solution works, but it's not ideal. If you put `console.log('Resubscribing')` inside the Effect, you'll notice that it resubscribes after every re-render. Resubscribing is fast, but it would still be nice to avoid doing it so often.

A better fix would be to move the `handleMove` function *inside* the Effect. Then `handleMove` won't be a reactive value, and so your Effect won't depend on a function. Instead, it will need to depend on `canMove` which your code now reads from inside the Effect. This matches the behavior you wanted, since your Effect will now stay synchronized with the value of `canMove`:

<Sandpack>

```js
import { useState, useEffect } from 'react';

export default function App() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [canMove, setCanMove] = useState(true);

  useEffect(() => {
    function handleMove(e) {
      if (canMove) {
        setPosition({ x: e.clientX, y: e.clientY });
      }
    }

    window.addEventListener('pointermove', handleMove);
    return () => window.removeEventListener('pointermove', handleMove);
  }, [canMove]);

  return (
    <>
      <label>
        <input type="checkbox"
          checked={canMove}
          onChange={e => setCanMove(e.target.checked)} 
        />
        The dot is allowed to move
      </label>
      <hr />
      <div style={{
        position: 'absolute',
        backgroundColor: 'pink',
        borderRadius: '50%',
        opacity: 0.6,
        transform: `translate(${position.x}px, ${position.y}px)`,
        pointerEvents: 'none',
        left: -20,
        top: -20,
        width: 40,
        height: 40,
      }} />
    </>
  );
}
```

```css
body {
  height: 200px;
}
```

</Sandpack>

Try adding `console.log('Resubscribing')` inside the Effect body and notice that now it only resubscribes when you toggle the checkbox (`canMove` changes) or edit the code. This makes it better than the previous approach that always resubscribed.

You'll learn a more general approach to this type of problem in [Separating Events from Effects.](/learn/separating-events-from-effects)

</Solution>

#### Fix a connection switch {/*fix-a-connection-switch*/}

In this example, the chat service in `chat.js` exposes two different APIs: `createEncryptedConnection` and `createUnencryptedConnection`. The root `App` component lets the user choose whether to use encryption or not, and then passes down the corresponding API method to the child `ChatRoom` component as the `createConnection` prop.

Notice that initially, the console logs say the connection is not encrypted. Try toggling the checkbox on: nothing will happen. However, if you change the selected room after that, then the chat will reconnect *and* enable encryption (as you'll see from the console messages). This is a bug. Fix the bug so that toggling the checkbox *also* causes the chat to reconnect.

<Hint>

Suppressing the linter is always suspicious. Could this be a bug?

</Hint>

<Sandpack>

```js App.js
import { useState } from 'react';
import ChatRoom from './ChatRoom.js';
import {
  createEncryptedConnection,
  createUnencryptedConnection,
} from './chat.js';

export default function App() {
  const [roomId, setRoomId] = useState('general');
  const [isEncrypted, setIsEncrypted] = useState(false);
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <label>
        <input
          type="checkbox"
          checked={isEncrypted}
          onChange={e => setIsEncrypted(e.target.checked)}
        />
        Enable encryption
      </label>
      <hr />
      <ChatRoom
        roomId={roomId}
        createConnection={isEncrypted ?
          createEncryptedConnection :
          createUnencryptedConnection
        }
      />
    </>
  );
}
```

```js ChatRoom.js active
import { useState, useEffect } from 'react';

export default function ChatRoom({ roomId, createConnection }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [roomId]);

  return <h1>Welcome to the {roomId} room!</h1>;
}
```

```js chat.js
export function createEncryptedConnection(roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ 🔐 Connecting to "' + roomId + '... (encrypted)');
    },
    disconnect() {
      console.log('❌ 🔐 Disconnected from "' + roomId + '" room (encrypted)');
    }
  };
}

export function createUnencryptedConnection(roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '... (unencrypted)');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room (unencrypted)');
    }
  };
}
```

```css
label { display: block; margin-bottom: 10px; }
```

</Sandpack>

<Solution>

If you remove the linter suppression, you will see a lint error. The problem is that `createConnection` is a prop, so it's a reactive value. It can change over time! (And indeed, it should--when the user ticks the checkbox, the parent component passes a different value of the `createConnection` prop.) This is why it should be a dependency. Include it in the list to fix the bug:

<Sandpack>

```js App.js
import { useState } from 'react';
import ChatRoom from './ChatRoom.js';
import {
  createEncryptedConnection,
  createUnencryptedConnection,
} from './chat.js';

export default function App() {
  const [roomId, setRoomId] = useState('general');
  const [isEncrypted, setIsEncrypted] = useState(false);
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <label>
        <input
          type="checkbox"
          checked={isEncrypted}
          onChange={e => setIsEncrypted(e.target.checked)}
        />
        Enable encryption
      </label>
      <hr />
      <ChatRoom
        roomId={roomId}
        createConnection={isEncrypted ?
          createEncryptedConnection :
          createUnencryptedConnection
        }
      />
    </>
  );
}
```

```js ChatRoom.js active
import { useState, useEffect } from 'react';

export default function ChatRoom({ roomId, createConnection }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, createConnection]);

  return <h1>Welcome to the {roomId} room!</h1>;
}
```

```js chat.js
export function createEncryptedConnection(roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ 🔐 Connecting to "' + roomId + '... (encrypted)');
    },
    disconnect() {
      console.log('❌ 🔐 Disconnected from "' + roomId + '" room (encrypted)');
    }
  };
}

export function createUnencryptedConnection(roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '... (unencrypted)');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room (unencrypted)');
    }
  };
}
```

```css
label { display: block; margin-bottom: 10px; }
```

</Sandpack>

It is correct that `createConnection` is a dependency. However, this code is a bit fragile because someone could edit the `App` component to pass an inline function as the value of this prop. In that case, its value would be different every time the `App` component re-renders, so the Effect might re-synchronize too often. To avoid this, you can pass `isEncrypted` down instead:

<Sandpack>

```js App.js
import { useState } from 'react';
import ChatRoom from './ChatRoom.js';

export default function App() {
  const [roomId, setRoomId] = useState('general');
  const [isEncrypted, setIsEncrypted] = useState(false);
  return (
    <>
      <label>
        Choose the chat room:{' '}
        <select
          value={roomId}
          onChange={e => setRoomId(e.target.value)}
        >
          <option value="general">general</option>
          <option value="travel">travel</option>
          <option value="music">music</option>
        </select>
      </label>
      <label>
        <input
          type="checkbox"
          checked={isEncrypted}
          onChange={e => setIsEncrypted(e.target.checked)}
        />
        Enable encryption
      </label>
      <hr />
      <ChatRoom
        roomId={roomId}
        isEncrypted={isEncrypted}
      />
    </>
  );
}
```

```js ChatRoom.js active
import { useState, useEffect } from 'react';
import {
  createEncryptedConnection,
  createUnencryptedConnection,
} from './chat.js';

export default function ChatRoom({ roomId, isEncrypted }) {
  useEffect(() => {
    const createConnection = isEncrypted ?
      createEncryptedConnection :
      createUnencryptedConnection;
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, isEncrypted]);

  return <h1>Welcome to the {roomId} room!</h1>;
}
```

```js chat.js
export function createEncryptedConnection(roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ 🔐 Connecting to "' + roomId + '... (encrypted)');
    },
    disconnect() {
      console.log('❌ 🔐 Disconnected from "' + roomId + '" room (encrypted)');
    }
  };
}

export function createUnencryptedConnection(roomId) {
  // A real implementation would actually connect to the server
  return {
    connect() {
      console.log('✅ Connecting to "' + roomId + '... (unencrypted)');
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room (unencrypted)');
    }
  };
}
```

```css
label { display: block; margin-bottom: 10px; }
```

</Sandpack>

In this version, the `App` component passes a boolean prop instead of a function. Inside the Effect, you decide which function to use. Since both `createEncryptedConnection` and `createUnencryptedConnection` are declared outside the component, they aren't reactive, and don't need to be dependencies. You'll learn more about this in [Removing Effect Dependencies.](/learn/removing-effect-dependencies)

</Solution>

#### Populate a chain of select boxes {/*populate-a-chain-of-select-boxes*/}

In this example, there are two select boxes. One select box lets the user pick a planet. Another select box lets the user pick a place *on that planet.* The second box doesn't work yet. Your task is to make it show the places on the chosen planet.

Look at how the first select box works. It populates the `planetList` state with the result from the `"/planets"` API call. The currently selected planet's ID is kept in the `planetId` state variable. You need to find where to add some additional code so that the `placeList` state variable is populated with the result of the `"/planets/" + planetId + "/places"` API call.

If you implement this right, selecting a planet should populate the place list. Changing a planet should change the place list.

<Hint>

If you have two independent synchronization processes, you need to write two separate Effects.

</Hint>

<Sandpack>

```js App.js
import { useState, useEffect } from 'react';
import { fetchData } from './api.js';

export default function Page() {
  const [planetList, setPlanetList] = useState([])
  const [planetId, setPlanetId] = useState('');

  const [placeList, setPlaceList] = useState([]);
  const [placeId, setPlaceId] = useState('');

  useEffect(() => {
    let ignore = false;
    fetchData('/planets').then(result => {
      if (!ignore) {
        console.log('Fetched a list of planets.');
        setPlanetList(result);
        setPlanetId(result[0].id); // Select the first planet
      }
    });
    return () => {
      ignore = true;
    }
  }, []);

  return (
    <>
      <label>
        Pick a planet:{' '}
        <select value={planetId} onChange={e => {
          setPlanetId(e.target.value);
        }}>
          {planetList.map(planet =>
            <option key={planet.id} value={planet.id}>{planet.name}</option>
          )}
        </select>
      </label>
      <label>
        Pick a place:{' '}
        <select value={placeId} onChange={e => {
          setPlaceId(e.target.value);
        }}>
          {placeList.map(place =>
            <option key={place.id} value={place.id}>{place.name}</option>
          )}
        </select>
      </label>
      <hr />
      <p>You are going to: {placeId || '???'} on {planetId || '???'} </p>
    </>
  );
}
```

```js api.js hidden
export function fetchData(url) {
  if (url === '/planets') {
    return fetchPlanets();
  } else if (url.startsWith('/planets/')) {
    const match = url.match(/^\/planets\/([\w-]+)\/places(\/)?$/);
    if (!match || !match[1] || !match[1].length) {
      throw Error('Expected URL like "/planets/earth/places". Received: "' + url + '".');
    }
    return fetchPlaces(match[1]);
  } else throw Error('Expected URL like "/planets" or "/planets/earth/places". Received: "' + url + '".');
}

async function fetchPlanets() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve([{
        id: 'earth',
        name: 'Earth'
      }, {
        id: 'venus',
        name: 'Venus'
      }, {
        id: 'mars',
        name: 'Mars'        
      }]);
    }, 1000);
  });
}

async function fetchPlaces(planetId) {
  if (typeof planetId !== 'string') {
    throw Error(
      'fetchPlaces(planetId) expects a string argument. ' +
      'Instead received: ' + planetId + '.'
    );
  }
  return new Promise(resolve => {
    setTimeout(() => {
      if (planetId === 'earth') {
        resolve([{
          id: 'laos',
          name: 'Laos'
        }, {
          id: 'spain',
          name: 'Spain'
        }, {
          id: 'vietnam',
          name: 'Vietnam'        
        }]);
      } else if (planetId === 'venus') {
        resolve([{
          id: 'aurelia',
          name: 'Aurelia'
        }, {
          id: 'diana-chasma',
          name: 'Diana Chasma'
        }, {
          id: 'kumsong-vallis',
          name: 'Kŭmsŏng Vallis'        
        }]);
      } else if (planetId === 'mars') {
        resolve([{
          id: 'aluminum-city',
          name: 'Aluminum City'
        }, {
          id: 'new-new-york',
          name: 'New New York'
        }, {
          id: 'vishniac',
          name: 'Vishniac'
        }]);
      } else throw Error('Unknown planet ID: ' + planetId);
    }, 1000);
  });
}
```

```css
label { display: block; margin-bottom: 10px; }
```

</Sandpack>

<Solution>

There are two independent synchronization processes:

- The first select box is synchronized to the remote list of planets.
- The second select box is synchronized to the remote list of places for the current `planetId`.

This is why it makes sense to describe them as two separate Effects. Here's an example of how you could do this:

<Sandpack>

```js App.js
import { useState, useEffect } from 'react';
import { fetchData } from './api.js';

export default function Page() {
  const [planetList, setPlanetList] = useState([])
  const [planetId, setPlanetId] = useState('');

  const [placeList, setPlaceList] = useState([]);
  const [placeId, setPlaceId] = useState('');

  useEffect(() => {
    let ignore = false;
    fetchData('/planets').then(result => {
      if (!ignore) {
        console.log('Fetched a list of planets.');
        setPlanetList(result);
        setPlanetId(result[0].id); // Select the first planet
      }
    });
    return () => {
      ignore = true;
    }
  }, []);

  useEffect(() => {
    if (planetId === '') {
      // Nothing is selected in the first box yet
      return;
    }

    let ignore = false;
    fetchData('/planets/' + planetId + '/places').then(result => {
      if (!ignore) {
        console.log('Fetched a list of places on "' + planetId + '".');
        setPlaceList(result);
        setPlaceId(result[0].id); // Select the first place
      }
    });
    return () => {
      ignore = true;
    }
  }, [planetId]);

  return (
    <>
      <label>
        Pick a planet:{' '}
        <select value={planetId} onChange={e => {
          setPlanetId(e.target.value);
        }}>
          {planetList.map(planet =>
            <option key={planet.id} value={planet.id}>{planet.name}</option>
          )}
        </select>
      </label>
      <label>
        Pick a place:{' '}
        <select value={placeId} onChange={e => {
          setPlaceId(e.target.value);
        }}>
          {placeList.map(place =>
            <option key={place.id} value={place.id}>{place.name}</option>
          )}
        </select>
      </label>
      <hr />
      <p>You are going to: {placeId || '???'} on {planetId || '???'} </p>
    </>
  );
}
```

```js api.js hidden
export function fetchData(url) {
  if (url === '/planets') {
    return fetchPlanets();
  } else if (url.startsWith('/planets/')) {
    const match = url.match(/^\/planets\/([\w-]+)\/places(\/)?$/);
    if (!match || !match[1] || !match[1].length) {
      throw Error('Expected URL like "/planets/earth/places". Received: "' + url + '".');
    }
    return fetchPlaces(match[1]);
  } else throw Error('Expected URL like "/planets" or "/planets/earth/places". Received: "' + url + '".');
}

async function fetchPlanets() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve([{
        id: 'earth',
        name: 'Earth'
      }, {
        id: 'venus',
        name: 'Venus'
      }, {
        id: 'mars',
        name: 'Mars'        
      }]);
    }, 1000);
  });
}

async function fetchPlaces(planetId) {
  if (typeof planetId !== 'string') {
    throw Error(
      'fetchPlaces(planetId) expects a string argument. ' +
      'Instead received: ' + planetId + '.'
    );
  }
  return new Promise(resolve => {
    setTimeout(() => {
      if (planetId === 'earth') {
        resolve([{
          id: 'laos',
          name: 'Laos'
        }, {
          id: 'spain',
          name: 'Spain'
        }, {
          id: 'vietnam',
          name: 'Vietnam'        
        }]);
      } else if (planetId === 'venus') {
        resolve([{
          id: 'aurelia',
          name: 'Aurelia'
        }, {
          id: 'diana-chasma',
          name: 'Diana Chasma'
        }, {
          id: 'kumsong-vallis',
          name: 'Kŭmsŏng Vallis'        
        }]);
      } else if (planetId === 'mars') {
        resolve([{
          id: 'aluminum-city',
          name: 'Aluminum City'
        }, {
          id: 'new-new-york',
          name: 'New New York'
        }, {
          id: 'vishniac',
          name: 'Vishniac'
        }]);
      } else throw Error('Unknown planet ID: ' + planetId);
    }, 1000);
  });
}
```

```css
label { display: block; margin-bottom: 10px; }
```

</Sandpack>

This code is a bit repetitive. However, that's not a good reason to combine it into a single Effect! If you did this, you'd have to combine both Effect's dependencies into one list, and then changing the planet would refetch the list of all planets. Effects are not a tool for code reuse.

Instead, to reduce repetition, you can extract some logic into a custom Hook like `useSelectOptions` below:

<Sandpack>

```js App.js
import { useState } from 'react';
import { useSelectOptions } from './useSelectOptions.js';

export default function Page() {
  const [
    planetList,
    planetId,
    setPlanetId
  ] = useSelectOptions('/planets');

  const [
    placeList,
    placeId,
    setPlaceId
  ] = useSelectOptions(planetId ? `/planets/${planetId}/places` : null);

  return (
    <>
      <label>
        Pick a planet:{' '}
        <select value={planetId} onChange={e => {
          setPlanetId(e.target.value);
        }}>
          {planetList?.map(planet =>
            <option key={planet.id} value={planet.id}>{planet.name}</option>
          )}
        </select>
      </label>
      <label>
        Pick a place:{' '}
        <select value={placeId} onChange={e => {
          setPlaceId(e.target.value);
        }}>
          {placeList?.map(place =>
            <option key={place.id} value={place.id}>{place.name}</option>
          )}
        </select>
      </label>
      <hr />
      <p>You are going to: {placeId || '...'} on {planetId || '...'} </p>
    </>
  );
}
```

```js useSelectOptions.js
import { useState, useEffect } from 'react';
import { fetchData } from './api.js';

export function useSelectOptions(url) {
  const [list, setList] = useState(null);
  const [selectedId, setSelectedId] = useState('');
  useEffect(() => {
    if (url === null) {
      return;
    }

    let ignore = false;
    fetchData(url).then(result => {
      if (!ignore) {
        setList(result);
        setSelectedId(result[0].id);
      }
    });
    return () => {
      ignore = true;
    }
  }, [url]);
  return [list, selectedId, setSelectedId];
}
```

```js api.js hidden
export function fetchData(url) {
  if (url === '/planets') {
    return fetchPlanets();
  } else if (url.startsWith('/planets/')) {
    const match = url.match(/^\/planets\/([\w-]+)\/places(\/)?$/);
    if (!match || !match[1] || !match[1].length) {
      throw Error('Expected URL like "/planets/earth/places". Received: "' + url + '".');
    }
    return fetchPlaces(match[1]);
  } else throw Error('Expected URL like "/planets" or "/planets/earth/places". Received: "' + url + '".');
}

async function fetchPlanets() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve([{
        id: 'earth',
        name: 'Earth'
      }, {
        id: 'venus',
        name: 'Venus'
      }, {
        id: 'mars',
        name: 'Mars'        
      }]);
    }, 1000);
  });
}

async function fetchPlaces(planetId) {
  if (typeof planetId !== 'string') {
    throw Error(
      'fetchPlaces(planetId) expects a string argument. ' +
      'Instead received: ' + planetId + '.'
    );
  }
  return new Promise(resolve => {
    setTimeout(() => {
      if (planetId === 'earth') {
        resolve([{
          id: 'laos',
          name: 'Laos'
        }, {
          id: 'spain',
          name: 'Spain'
        }, {
          id: 'vietnam',
          name: 'Vietnam'        
        }]);
      } else if (planetId === 'venus') {
        resolve([{
          id: 'aurelia',
          name: 'Aurelia'
        }, {
          id: 'diana-chasma',
          name: 'Diana Chasma'
        }, {
          id: 'kumsong-vallis',
          name: 'Kŭmsŏng Vallis'        
        }]);
      } else if (planetId === 'mars') {
        resolve([{
          id: 'aluminum-city',
          name: 'Aluminum City'
        }, {
          id: 'new-new-york',
          name: 'New New York'
        }, {
          id: 'vishniac',
          name: 'Vishniac'
        }]);
      } else throw Error('Unknown planet ID: ' + planetId);
    }, 1000);
  });
}
```

```css
label { display: block; margin-bottom: 10px; }
```

</Sandpack>

Check the `useSelectOptions.js` tab in the sandbox to see how it works. Ideally, most Effects in your application should eventually be replaced by custom Hooks, whether written by you or by the community. Custom Hooks hide the synchronization logic, so the calling component doesn't know about the Effect. As you keep working on your app, you'll develop a palette of Hooks to choose from, and eventually you won't need to write Effects in your components very often.

</Solution>

</Challenges>
