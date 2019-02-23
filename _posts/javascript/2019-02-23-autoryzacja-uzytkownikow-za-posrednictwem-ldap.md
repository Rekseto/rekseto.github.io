---
layout: post
title:  "Autoryzacja użytkowników w Node.js za pośrednictwem LDAP"
author: Rekseto
date:   2019-02-24 17:20:00 +0200
categories: javascript
---

W związku z moim ostatnim zleceniem, w którym zetknąłem się z integracją mojej aplikacji z infrastruktura firmy opartej o Windows Server i usługę Active Directory. Byłem zmuszony poznać sposób na integrację autoryzacji mojej aplikacji z stworzonymi już użytkownikami Active Directory. Opiera się o LDAP, jako że nie znalazłem żadnego polskiego źródła mówiącego na ten temat, postanowiłem sam napisać posta, w którym bliżej przedstawię jak wygląda praca z Active Directory w node.js.

## Usługi Katalogowe a Autoryzacja

Każda usługa katalogowa stanowi pewnego rodzaju bazę danych, która przechowuje użytkowników czy inne obiekty naszej sieci. To pozwala logicznie i precyzyjnie opisywać oraz zarządzać relacjami w danej sieci. Jako że nas interesuje autoryzacja użytkowników z pomocą LDAP, będziemy odpytywać serwer czy dane poświadczenia logowania znajdują odzwierciedlenie w naszym ekosystemie. Sam klient LDAP komunikuje się z serwerem za pomocą 5 operacji:

- **bind** -  Uwierzytelnianie użytkownika
- **unbind** - Zakończenie sesji.
- **search** - Wyszukiwanie danego zasobu
- **add** - Dodawanie nowego zasobu
- **delete** - Usuniecie zasobu
- **modify** - Modyfikacja zasobu

W naszym przypadku będziemy korzystać tylko z bind, ale nic nie stoi na przeszkodzie by nasza aplikacja ściślej współpracowała z serwerem LDAP.

## Autoryzacja Client <-> Aplikacja

Oczywistym faktem jest że nie chcemy by użytkownik po przejściu na każda podstronę na nowo musiał wpisywać swoje dane, dlatego też zwykle używamy tu autoryzacji opartej o sesje albo o tokeny JWT, i właśnie tą drugą metodę spróbujemy zintegrować z naszą usługa katalogowa. W tym celu musimy przygotować sobie środowisko bazo danowe w moim przypadku będzie to baza MongoDB ze względu na prostotę. Otóż za każdym razem kiedy użytkownik będzie chciał się dostać do naszego serwisu prześle on swoje poświadczenia (nazwę i hasło), które następnie prześlemy do serwera LDAP. Po otrzymaniu pozytywnej odpowiedzi wstawimy (jeżeli nie istnieje) następujący obiekt do naszej bazy.

```javascript
{
    username: "test",
    secret: getRandomString(16)
}
```

Wstawienie takiego rekordu do bazy spowoduje, że w każdym momencie kiedy przyjdzie do sprawdzania poprawności tokena, sięgamy do bazy danych wyszukujemy użytkownika o danej nazwie, następnie pobieramy unikalną wartość secret która będzie stanowić pewnego rodzaju segment całego sekretu którym będzie szyfrowany każdy token. Oczywiście moglibyśmy używać tylko jednej wspólnej wartości sekret do szyfrowania tokena JWT, ale wtedy:

1. Tracimy na bezpieczeństwie
2. Odbieramy sobie możliwość anulowania wszystkich tokenów dla jednego usera

dlatego też zaimplementujemy taką wersje tokenizacji.

## Biblioteki do Node.js

Jeżeli chodzi o biblioteki pozwalające kontaktować się nam z serwerem LDAP, to ja osobiście preferuję [ldapjs](https://github.com/joyent/node-ldapjs>), która stanowi chyba najprzystępniejszą jaką znalazłem, niestety, sama biblioteka powstała stosunkowo dosyć dawno i nie implementuje ona natywnie Promises, które ułatwiają nam pracę przy asynchronicznym kodzie, dlatego też użyjemy [promised-ldap](<https://www.npmjs.com/package/promised-ldap>).

## Kod

Dla mnie najszybszym sposobem napisania takiego kodu będzie skorzystanie z [inra-http-boilerplate](https://github.com/project-inra/inra-http-boilerplate).

```javascript
export default function(config) {
  const logger = console;
  // Tutaj ustawiam za logger domyślną konsole ale nic nie stoi na przeszkodzie wykorzystaniu jakieś biblioteki pokroju winston

  const database = initDatabase({logger}, config); // Tutaj tworzę obiekt bazy

  const server = initServer({logger, database}, config); 
  // Jako że wszystkie elementy naszej aplikacji korzystaja z bazy danych to przekazujemy ją do serwera

  return new Promise((resolve, reject) => {
    database
      .connect()
      .then(res => {
        server.run(config.SERVER_PORT, function() {
          logger.log(`Server running on ${config.SERVER_PORT}`);

          resolve(server);
        });
      })
      .catch(err => {
        logger.error("A critial error occured on startup");
        logger.error(err);

        reject(err);
      });
  });
}

```



Oto zasadniczo najważniejsza cześć mojego App.js, gdzie jak widać tworzę obiekt bazy której będziemy używać. inra-http pozwala mi na łatwe zarządzanie kodem, zasadniczo wszystko rozbijemy na:

- Model
- Service
- Middleware
- Route (Ścieżka)

### Model Użytkownika

Jak już wcześniej stwierdziłem nasz model będzie posiadał prostą strukture.

```javascript
export default mongoose => {
  const Schema = mongoose.Schema;

  const User = new Schema({
    username: {type: String, required: true, unique: true}, // Pole username musi być unikalne
    secret: {type: String, required: true} // Secret nigdy nie powinien być pusty
  });

  return {schema: User, modelName: "User"};
};
```

Model posiada tylko pole username (pozwoli nam identyfikować użytkownika), i secret którego będziemy używać do szyfrowania tokenów JWT.

### Service Auth

Service jest elementem naszej aplikacji gdzie znajdują się funkcje odpowiedzialne za operacje na danych typu: tworzenie użytkownika, pobieranie wszystkich użytkowników z bazy, usuwanie użytkowników. Funkcje te będą później wykorzystywane w obsługiwaniu ścieżek.

```javascript
import LdapClient from "promised-ldap"; 
import InternalServerError from "../errors/InternalServerError";
import {getRandomString} from "../../utils/cryptoUtils";
import {generateToken} from "../../utils/authUtils";

export default (database, logger) => {
  const {User} = database.models; // Model Usera który stworzyliśmy

  return {
    async createUser({username}) { // Funkcja odpowiadająca za tworzenie użytkownika
      try {
        const secret = getRandomString(16); // 16-znakowy sekret
        const user = await User.create({
          username,
          secret
        }); // Bezpośrednie tworzenie użytkownika

        return user;
      } catch (error) {
        throw error;
      }
    },

    async login({username, password}) { 
// Funkcja odpowiadająca logowaniu przyjmuje username i password
      try {
        if (!password) { 
// Jeżeli hasło nie zostało podane to wyrzucamy błąd.. (Aczkolwiek jeżeli chcemy by użytkownik mógł logować się bez hasła to nic nie stoi na przeszkodzie w usunieciu tego fragmentu kodu)
          throw new InternalServerError();
        }

        const client = new LdapClient({ // Tutaj tworzymy Klienta LDAP 
          url: "ldap://test.local" 
          // Tutaj podajemy domene na której postawiony jest nasz serwer LDAP
        }); 

        await client.bind(username, password);
          // Z dokumentacji jasno wynika że operacja bind jest odpowiedzialna za uwierzytelnianie. Jeżeli wystąpi błąd to zostanie on wyrzucony do wyższej warstwy naszej aplikacji

        await client.unbind();
        // Jeżeli już nie potrzebujemy korzystać z LDAP to możemy zakończyć połączenie
        let userRecord = await User.findOne({username}); // Wyszukujemy danego użytkownika w bazie

        if (!userRecord) {
          // Tutaj tworzymy użytkownika w razie gdyby nie było go jeszcze w bazie
          userRecord = await this.createUser({username});
        }

        const token = generateToken({
          // No i na samym końcu możemy wygenerować token JWT
          id: userRecord._id, // Z id będziemy potem korzystać przy odczytywaniu tokena
          secret: userRecord.secret 
        });

        return token;
      } catch (error) {
        logger.log(error);
        throw error;
      }
    }
  };
};
```

Powyższy kod zawiera tylko 2 proste funkcje.Pierwsza odpowiedzialna za stworzenie użytkownika. Druga zajmująca się logowaniem.

Użycie LDAP jest tutaj jak widać banalne, wystarczy przesłać za pomocą funkcji **bind** nazwę użytkownika oraz jego hasło, jeżeli wystąpi błąd to znaczy, że użytkownika nie ma w Active Directory albo hasło jest niepoprawne. Jeżeli wszystko przebiegło pomyślnie to zwracamy token. Generowanie tokena realizujemy poprzez wcześniej stworzona funkcje pomocniczą generateToken, znajduję się ona w authUtils.js

```javascript
/**
 * Generates a JSON Web Token for a given user id and user secret.
 *
 * @param   {Mongoose}  record      User record from database
 * @param   {number}    expiresIn   Expiration date (in seconds)
 *
 * @return  {string}                Signed JSON Web Token
 */
export function generateToken(record, expiresIn = 86400) {
  const serverSecret = process.env.AUTH_SECRET; // Pobieramy sekret z zmiennej środowiskowej
  const tokenSecret = `${record.secret}@${serverSecret}`; 

 
   // Tutaj korzystamy z funkcji bilbioteki jsonwebtoken
  return jwt.sign({id: record.id}, tokenSecret, {expiresIn}); 
}

```

Łączymy sekret unikalny dla każdego użytkownika z globalnym tokenem, w przypadku zmiany któregokolwiek z nich unieważniamy tokeny oparte o takie połączenie. Jeżeli zmienimy tylko sekret użytkownika, to unieważnimy wszystkie tokeny użytkownika. W przypadku zmiany AUTH_SECRET unieważnimy każdy token.



### Middleware isAuthorized

Middleware można nazwać funkcja która wywołujemy przed obsłużeniem scieżki, tutaj akurat wykorzystamy middleware do sprawdzenia tokena który zostaje nam przesłany w nagłówku żądania. Jeżeli token jest poprawny to żądanie zostanie wykonane tak jak zdefiniowaliśmy w scieżce, jeżeli nie to wyrzucimy błąd który przejmie nasz serwer.

```javascript
@middleware()
class isAuthorizedMiddleware {
  constructor({database}) {
    this.models = database.models;
  }

  /**
   * Tries to extract JWT token from header/query.
   *
   * @param  {Context}  ctx
   * @param  {Function} next
   * @return {Promise}
   */
  async before(ctx, next) {
    this.token = extractToken(ctx);

    if (!this.token) {
    // Token nie został wysłany w Headerze 
      throw new NotAllowedError("Cannot extract token from context");
    }
  }
  /**
   * Checks if provided JWT is valid and populates context with user data.
   *
   * @param  {Context}  ctx
   * @param  {Function} next
   * @return {Promise}
   */
  async handle(ctx, next) {
    const data = await this.fetchUser(); // Pobieramy użytkownika
    const secret = `${data.secret}@${process.env.AUTH_SECRET}`;
 // Pobieramy sekret użytkownika, a następnie łaczymy go z globalnym sekretem.
    
// Deszyfrujemy token za pomocą sekretu 
    return jwt.verify(this.token, secret, (error, decoded) => {
       
      if (error) { 
// Jeżeli coś poszło nie tak to znaczy, że token jest unieważniony
        throw new NotAllowedError(error.message);
      }

        
      // Przypisujemy do obiektu ctx użytkownika i zdekodowane dane z jwt
      ctx.state.user = data;
      ctx.state.jwt = decoded;

      return next();
    });
  }

  /**
   * Extracts basic user data from database for further usage.
   *
   * @return {Promise|Object}
   */
  async fetchUser() {
    try {
      const {User} = this.models; 
      const {payload} = decodeToken(this.token); 
      // dekodujemy token i uzyskujemy dostęp do wcześniej przypisanej wartości id
        
      return User.findOne({
        _id: payload.id
      });
    } catch (error) {
      throw error;
    }
  }
}
```



### Route Auth

Route (Ścieżka) to cześć aplikacji w której odbieramy żądania i decydujemy się co z nimi zrobić.

```javascript
import compose from "koa-compose";
import controller, {get, post, del, put} from "inra-server-http/dest/router";
import authServices from "../services/authServices";

@controller("/auth")
export default class AuthRouter {
  constructor(dependencies) {
    this.database = dependencies.database;
    this.logger = dependencies.logger;

    this.authServices = authServices(this.database, this.logger);
  }

  @post("/login")
  async login(ctx) {
    try {
      const {username, password} = ctx.request.body;
      // Wyciągamy 2 zmienne z ciała zapytania

      // Dalej wywołujemy funkcje z authServices która odpowida z logowanie
      const data = await this.authServices.login({
        username,
        password
      });

      // Wszystko jest ok, autoryzacja przeszła bez błędu i mamy swój token
      ctx.body = {
        success: true,
        token: data
      };
    } catch (error) {
      // Jeżeli funkcja wyrzuciła błąd to przekazujemy go dalej do Inry
      throw error;
    }
  }

  @get("/verify", function() {
    return compose([this.isAuthorized()]);
  })
  async verify(ctx) {
    // Middleware nie wyrzuciło nam błędu, więc wszystko jest ok
    ctx.body = {
      success: true
    };
  }
}

```

Powyższy plik tak naprawdę odpowiada tylko za skorzystanie z authServices, oraz przesłanie odpowiedzi zwrotnej.

## Podsumowanie

Nasza aplikacja jest już w pełni zintegrowana z Active Directory. Za każdym razem kiedy API będzie musiało zweryfikować pewne poświadczenia, wywołamy zapytanie do serwera LDAP. Po pozytywnej odpowiedzi wyszukujemy czy użytkownik jest w naszej bazie jeżeli jest to wyciągamy od niego wartość sekret i generujemy na jej podstawie token. Jeżeli go nie ma to dodajemy rekord do bazy, następnie generujemy token. W razie potrzeby unieważnienia tokenów możemy zmienić sekretną wartość każdego użytkownika albo jedną wspólną dla wszystkich tokenów wartość AUTH_SECRET. Samo wdrożenie naszej aplikacji do infrastruktury opartej o LDAP nie jest zatem czymś trudnym, a czasem takie rozwiązanie pozwoli nam łatwo wdrożyć wewnątrz firmową aplikacje.

To tyle na dzisiaj, mam nadzieje że w jakiś sposób pomogłem wam zintegrować wasze aplikacje z serwerami LDAP. Kod aplikacji dostępny jest na moim [githubie](https://github.com/) pod tym [adresem](https://github.com/Rekseto/auth-ldap-example).

