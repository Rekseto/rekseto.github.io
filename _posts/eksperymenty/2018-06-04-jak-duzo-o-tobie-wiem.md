---
layout: post
title:  "Jak dużo mogę wiedzieć o Tobie?"
author: Rekseto
date:   2018-06-04 23:25:00 +0200
categories: eksperymenty inne javascript
---


Ostatnimi czasy podczas szumu związanego z **RODO** i różnymi kradzieżami tożsamości/danych, ludzie coraz bardziej martwią się oto ile informacji można o nich wyciągnąć bez ich wiedzy. W dzisiejszym artykule przedstawię kilka informacji które bez większych problemów pozwalają mi na stworzenie profilu użytkownika który odwiedza moją stronę. Do napisania tego bloga zainspirował mnie post mojego kolegi, znajdziecie go pod tym [adresem](https://laniewski.me/security/2018/02/17/identyfikacja-uzytkownika-na-podstawie-odcisku-przegladarki.html).


[1. Łatwo dostępne informacje](#1-łatwo-dostępne-informacje)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.1 System operacyjny](#11-system-operacyjny)
	
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.2 DuckTyping jako metoda rozpoznawania przegląderek](#12-ducktyping-jako-metoda-rozpoznawania-przeglądarek)
	
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.3 Sprawdzanie jakim językiem prawdopodobnie nasz użytkownik biegle operuje](#13-sprawdzanie-jakim-językiem-prawdopodobnie-nasz-użytkownik-biegle-operuje)
	
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.4 Jakimi językami nasz użytkownik jeszcze się posługuje?](#14-jakimi-językami-nasz-użytkownik-jeszcze-się-posługuje)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.5 Sprawdzanie liczby rdzeni procesora](#15-sprawdzanie-liczby-rdzeni-procesora)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.6 Laptop czy PC](#16-laptop-czy-pc)

[2. Trudniej dostępne informacje](#2-trudniej-dostępne-informacje)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[2.1 GPU użytkownika](#21-gpu-użytkownika)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[2.2 Czy nasz użytkownik jest zalogowany na Facebooku?](#22-czy-nasz-użytkownik-jest-zalogowany-na-facebooku)

###  1. Łatwo dostępne informacje
Podstawowymi informacjami jakie mogę uzyskać jest rodzaj twojej przeglądarki i systemu operacyjnego. Przeglądarka
opakowuje te informacje w zmienne i umieszcza do swobodnego pobrania przez Web APIs.
#### 1.1 System operacyjny
```javascript
const system = () => {
    let OSName = 'Other OS';
    if (navigator.appVersion.indexOf('Win') != -1) OSName = 'Windows';
    if (navigator.appVersion.indexOf('Mac') != -1) OSName = 'MacOS';
    if (navigator.appVersion.indexOf('X11') != -1) OSName = 'UNIX';
    if (navigator.appVersion.indexOf('Linux') != -1) OSName = 'Linux';
    return OSName;
};
```
Kod wyżej dla mojego systemu którym jest Windows 8.1 zwraca Windows. Jak widać mamy tu do czynienia z obiektem "navigatora" w którym umieszczona jest zmienna o niezbyt tajemniczej nazwie appVersion. Ale czym ten navigator jest? Kierując się [MDN](https://developer.mozilla.org/pl/)
  
>The `**Navigator**` interface represents the state and the identity of the user agent. It allows scripts to query it and to register themselves to carry on some activities.

Krótko mówiąc mamy do czynienia z obiektem pozwalającym zidentyfikować tak zwany user agent czyli obiekt identyfikujący naszą przeglądarke i system operacyjny.

więc jak wygląda te całe appVersion?
```5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36```

Oto przykładowe appVersion, jak widać identyfikowanie z jakim systemem mamy do czynienia wydaje się być dużo prostsze niż rozpoznanie przeglądarki bo widnieją tu przynajmniej 2!

#### 1.2 DuckTyping jako metoda rozpoznawania przeglądarek
Jedna z znanych metod rozpoznawania przeglądarki jest tak zwany DuckTyping.
No dobra dobra ale czymże to jest? Znowu posłużę się przykładem kodu:
```javascript
const browser = () => {
    if (!!window.chrome && !!window.chrome.webstore) {
        return 'Google Chrome';
    } else if (typeof InstallTrigger !== 'undefined') {
        return 'Firefox';
    } else if (
        (!!window.opr && !!opr.addons) ||
        !!window.opera ||
        navigator.userAgent.indexOf(' OPR/') >= 0
    ) {
        return 'Opera';
    } else if (
        /constructor/i.test(window.HTMLElement) ||
        (function(p) {
            return p.toString() === '[object SafariRemoteNotification]';
        })(!window['safari'] || safari.pushNotification)
    ) {
        return 'Safari';
    } else if (!!window.StyleMedia) {
        return 'Internet Explorer';
    }
};
```
Na pozór ten kod jest strasznie zagmatwany ale postaram się wyjaśnić krok po kroku co tu się dzieje.
Jak wiemy przeglądarki pomimo wciąż powstających standardów różnią się masą rzeczy (choćby wsparciem), ale posiadając odpowiednią wiedzę jesteśmy w stanie na podstawie tych różnic rozpoznać z jaka przeglądarka mamy do czynienia.

Pierwsza przeglądarka nie powinna stanowić problemu, sprawdzamy czy obiekt window zawiera w sobie wartość Chrome (ustawiana przez przeglądarkę) a następnie czy zawiera w sobie WebStore czyli magazyn dodatków.

Z Firefoxem mamy odrobione mniejsza pule informacji do sprawdzenia, tutaj akurat sprawdzamy funkcje od instalowania dodatków więc można powiedzieć że jest podobnie jak w Chrome.

Jadąc dalej nie mamy nadal jakiegoś wyzwania więc pominę szerszy opis Opery bo opiera się dokładnie na tej samej zasadzie z tą różnica że tu sprawdzamy dodatkowo user agent w poszukiwaniu Opery.

Z Safari wykorzystujemy typowe różnice w implementacji API.

I wisienka na torcie sprawdzanie czy mamy do czynienia z IE, wykorzystujemy kolejną różnicę w implementacji API i sprawdzamy istnienie wartości która AFAIK ma prawo egzystować tylko w IE ewt. w Edge.

Uogólniając DuckTyping to metoda rozpoznawania np. przeglądarek na podstawie zmiennych i różnic w implementacji WebAPIs.

#### 1.3 Sprawdzanie jakim językiem prawdopodobnie nasz użytkownik biegle operuje.
No własnie jak rozpoznać czy nasz użytkownik jest Polakiem czy może Francuzem?
Najczęściej ludzie w przeglądarce za język ustawiony mają swój własny ojczysty co pozwala na szybka identyfikacje, bo jak możecie się domyślić mamy do tej informacji swobodny dostęp:

```javascript
const lang = () => navigator.language;
```

Czyż to nie proste? Osobiście uważam że wszelkie przypadki typu: "Zmienię język w przeglądarce żeby go lepiej poznać", "Mieszkam za granica i taki mam język ustawiony", możemy potraktować jako niezbyt interesująca nas nisze aczkolwiek ostatni typ ja bym zaliczył nadal do biegłego mówcy.

#### 1.4 Jakimi językami nasz użytkownik jeszcze się posługuje?

Obiekt navigatora udostępnia nam jeszcze ciekawa informacje o preferowanych językach naszego użytkownika.

```js
const lang = () => navigator.languages;
```
Przykładowy wynik:
```["pl-PL", "pl", "en-US", "en"]```

#### 1.5 Sprawdzanie liczby rdzeni procesora

Z pozoru zadanie wydaję się trudniejsze, czyż nie?
```javascript
const logicalCores = () => window.navigator.hardwareConcurrency
```

No cóż czasem po prostu dane są bardzo łatwo dostepne ;).

#### 1.6 Laptop czy PC
Jednym z nowszych WebAPIs pozwalającym nam ocenić czy nasz użytkownik przegląda nasza stronę z komputera
czy też może z laptopa jest "Battery Status API", najlepiej wyjaśnia się kod więc oto on:
```js
navigator.getBattery().then(function(battery) {
  function updateAllBatteryInfo(){
    updateChargeInfo();
    updateLevelInfo();
    updateChargingInfo();
    updateDischargingInfo();
  }
  updateAllBatteryInfo();

  battery.addEventListener('chargingchange', function(){
    updateChargeInfo();
  });
  function updateChargeInfo(){
    console.log("Battery charging? "
                + (battery.charging ? "Yes" : "No"));
  }

  battery.addEventListener('levelchange', function(){
    updateLevelInfo();
  });
  function updateLevelInfo(){
    console.log("Battery level: "
                + battery.level * 100 + "%");
  }

  battery.addEventListener('chargingtimechange', function(){
    updateChargingInfo();
  });
  function updateChargingInfo(){
    console.log("Battery charging time: "
                 + battery.chargingTime + " seconds");
  }

  battery.addEventListener('dischargingtimechange', function(){
    updateDischargingInfo();
  });
  function updateDischargingInfo(){
    console.log("Battery discharging time: "
                 + battery.dischargingTime + " seconds");
  }

});
```
Przykład został wprost wyjęty z [dokumentacji](https://developer.mozilla.org/en-US/docs/Web/API/Battery_Status_API).

No dobra ale co to ma wspólnego z laptopem czy PC, odpowiadając na te pytanie podam przykładowy wynik dla PC

```
Battery charging? Yes
Battery level: 100%
Battery charging time: 0 seconds
Battery discharging time: Infinity seconds
```

Elementem kluczowym jest tu chargning time i discharging time które odpowiednio zwracają 0 sekund i nieskończoność sekund. Natomiast w przypadku laptopów chargning time / discharging time i battery level  będą się odpowiednio zmieniać w czasie lub będą wskazywać jakieś inne wartości.



### 2. Trudniej dostępne informacje
Tę cześć wpisu poświecę rzadziej sprawdzanym informacją, po części przez trudność dostępu do nich ale też ze względu na niską wartość w diagnostyce działania aplikacji sieciowych. Ich zastosowanie leży głównie przy metodach [Phishingu](https://pl.wikipedia.org/wiki/Phishing), albo w celu śledzenia użytkownika.

#### 2.1 GPU użytkownika
```js
const gpu = () => {
    const canvas = document.getElementById("gl");
    try {
        gl = canvas.getContext("experimental-webgl");
        gl.viewportWidth = canvas.width;
        gl.viewportHeight = canvas.height;
    } catch (e) {}
    if (gl) {
        const extension = gl.getExtension('WEBGL_debug_renderer_info');
        return gl.getParameter(extension.UNMASKED_RENDERER_WEBGL)
    } else return "Not supported";

}
 ```
Wraz z pojawieniem się WebGLa mamy dostęp do ciekawych informacji, no tak ale do wyciągnięcia karty graficznej naszego użytkownika potrzebujemy 2 rzeczy: przeglądarki która wspiera WebGLa i elementu canvas który ma ustawiony kontekst na eksperymentalny webGL. Dalej zadanie już jest proste, mamy dostęp do parametrów z których wyciągamy renderer WebGLa.
Przykładowy wynik:

```ANGLE (Intel(R) HD Graphics 530 Direct3D11 vs_5_0 ps_5_0)```

Jak widać wyciągniecie tej informacji zmusza nas do umieszczenia specjalnego elementu na stronie co nie wydaje się być zbyt wygodne w jakiejkolwiek diagnostyce takich jak Google Analitics, aczkolwiek nadal jest to informacja z której mogą czerpać ludzie którzy np. chcą wcisnąć niewinnym użytkownikom kartę graficzna za typowy "bezcen",
albo do analizy specyfikacji naszego komputera (Ciekawie to brzmi w kontekście sprawdzenia czy nasze GPU nadaje się do kopania kryptowalut podczas przeglądania stron)

#### 2.2 Czy nasz użytkownik jest zalogowany na Facebooku?

I tutaj dochodzimy do informacji która powinna was zaniepokoić, bo czemu niby mielibyśmy dostep do sprawdzenia czy ktoś jest zalogowany na facebooku? Niestety albo stety sposób na to jest brudny tak samo jak wyciąganie informacji o GPU.

```js
(function(){
const url = "https://www.facebook.com/login.php?next=https%3A%2F%2Fwww.facebook.com%2Ffavicon.ico%3F_rdr%3Dp"
const img = document.createElement('img');
img.src = url;

img.onload = function() {
console.log('zalogowany');
}
img.onerror = function() {
console.log('niezalogowany');
}
}());
```
Wyżej przedstawione [IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) wykorzystuje pewien specyficzny link do favicona w momencie kiedy jesteśmy zalogowani wejście na ten adres spowoduje normalnie pobranie się favicona tak jak to robi strona faceeboka lecz kiedy nie jesteśmy zostajemy przekierowani na pełna strone logowania więc załadowany url nie może być obrazkiem a stworzony element wyrzuca błąd.

Bardzo niebezpieczna informacja mogąca służyć phisingowi, bo jeżeli użytkownik nie jest zalogowany oszust może umiejetnie podstawić okienko logowania które da mu dostep do jego facebooka ot cała filozofia.

### 3. Ochrona

Jeżeli chcemy się bronić przed zbieraniem takich danych przez administratorów stron które odwiedzamy, zalecam stosowanie różnych dodatków do przeglądarek które np.

-  Odpalaja javascript na zaufanych przez nas stronach
- Wprowadzaja chaos informacji do obiektów udostepnianych przez przeglądarke
- Nie wchodzenie na strony o dziwnej opinii


Ale zalecam o pamiętaniu że niektóre z tych informacji służą administratorom do celów czysto diagnostycznych takich jak to czy nasza przeglądarka obsługuje poprawnie technologie użyte na stronie lub czy odpowiednio szybko się ładuje czy też przystosowaniu języka strony do naszych oczekiwań.





