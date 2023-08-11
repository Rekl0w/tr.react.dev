---
title: React DOM API'leri
---

<Intro>

`react-dom` paketi, sadece tarayıcı DOM ortamında çalışan web uygulamaları için desteklenen yöntemleri içerir. React Native için desteklenmezler.

</Intro>

---

## API'ler {/*apis*/}

Bu API'ler bileşenlerinizden içe aktarılabilirler. Nadiren kullanılırlar:

* [`createPortal`](/reference/react-dom/createPortal) alt bileşenleri DOM ağacındaki farklı bir bölüme render etmenizi sağlar.
* [`flushSync`](/reference/react-dom/flushSync) React'i bir state güncellemesini hemen uygulamaya zorlayarak senkronize şekilde DOM'u güncellemenizi sağlar.

---

## Giriş noktaları {/*entry-points*/}

`react-dom` paketi iki ek giriş noktası sağlar:

* [`react-dom/client`](/reference/react-dom/client) React bileşenlerini istemcide (tarayıcıda) render etmek için API'ler içerir.
* [`react-dom/server`](/reference/react-dom/server) React bileşenlerini sunucuda oluşturmak için API'ler içerir.

---

## Kullanımdan kaldırılmış API'ler {/*deprecated-apis*/}

<Deprecated>

Bu API'ler React'in gelecekteki bir ana sürümünde kaldırılacaktır.

</Deprecated>

* [`findDOMNode`](/reference/react-dom/findDOMNode) bir sınıf bileşeni öğesine karşılık gelen en yakın DOM düğümünü bulur.
* [`hydrate`](/reference/react-dom/hydrate) sunucu HTML'inden oluşturulan DOM'a bir ağaç bağlar. [`hydrateRoot`](/reference/react-dom/client/hydrateRoot) ile değiştirilmiştir.
* [`render`](/reference/react-dom/render) bir ağacı DOM'a bağlar. [`createRoot`](/reference/react-dom/client/createRoot) ile değiştirilmiştir.
* [`unmountComponentAtNode`](/reference/react-dom/unmountComponentAtNode) bir ağacı DOM'dan kaldırır. [`root.unmount()`](/reference/react-dom/client/createRoot#root-unmount) ile değiştirilmiştir.

