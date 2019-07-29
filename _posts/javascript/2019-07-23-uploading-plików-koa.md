---
layout: post
title: "Uploading plików z koa.js"
author: Rekseto
date: 2019-07-29 15:10:00 +0200
categories: [javascript]
---

Przy wykonywaniu mojego ostatniego zlecenia, doszedłem do wniosku, że uploading plików na serwer przy użyciu busboy nie jest wcale taki jasny i prosty do wykonania, dlatego tym się dzisiaj zajmiemy.

## Konfiguracja serwera http

Na samym poczatku przygotujmy sobie przykładowy serwer http. Paczki których będziemy używać to:

- koa
- koa-router
- busboy
- koa-ejs

```typescript
import * as Koa from "koa";
import * as Router from "koa-router";

//@ts-ignore
import * as koaEjs from "koa-ejs";
import * as path from "path";

import routes from "./routes";

const app = new Koa();

koaEjs(app, {
  root: path.join(__dirname, "templates"),
  layout: false,
  viewExt: "ejs"
});

const router: Router = new Router();

routes(router);

app.use(router.routes());

app.listen(3000);
```

## Szablon html

Dla celów pokazowych w podstawowej konfiguracji serwera zainicjalizowałem moduł odpowiedzialny za renderowanie szablonów, stwórzmy więc przykładowy formularz.

```html
<!DOCTYPE html>
<html class="no-js" lang="">
  <head>
    <meta charset="utf-8" />
    <title>File uploading</title>
    <meta name="description" content="" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>

  <body>
    <!--[if IE]>
      <p class="browserupgrade">
        You are using an <strong>outdated</strong> browser. Please
        <a href="https://browsehappy.com/">upgrade your browser</a> to improve
        your experience and security.
      </p>
    <![endif]-->

    <form action="/" method="POST" enctype="multipart/form-data">
      <div class="inputGroup">
        <label for="file">Wybierz plik</label>
        <input type="file" name="file" id="file" />
      </div>

      <button type="submit">Prześlij</button>
    </form>
  </body>
</html>
```

### Scieżka z szablonem

Nasza ścieżka GET / ma zwracać nasz szablon

```typescript
import * as Router from "koa-router";

export default function(router: Router) {
  router.get("/", async ctx => {
    await ctx.render("uploadImage");
  });
```

## Uploading plików

Po wszystkich sprawach związanych z konfiguracją fundamentów naszego projektu możemy w końcu zabrać się za nasz gwóźdź programu czyli uploading plików, w tym celu tworzymy ścieżkę POST / która będzie odpowiedzialna za odbiór plików z utworzonego wcześniej przez nas formularza. Jako że będziemy korzystać z biblioteki [busboy](https://github.com/mscdex/busboy) zróbmy to co nakazuje nam dokumentacja, z tą różnicą że nasz kod odpowiedzialny za parsowanie nadchodzących danych zamkniemy w Promise.

```typescript
router.post("/", async ctx => {
  const { headers } = ctx;

  if (headers["content-length"] / (1024 * 1000) > 10) {
    throw new Error("Too big file");
    // Ograniczmy wielkość plików które można nam przesłać
  }

  const uploading = new Promise((resolve, reject) => {
    const busboy = new Busboy({ headers });
    const streams: Promise<any>[] = []; // Tu będą strumienie z zapisami naszych plików
    const parsingResult: IParsingResult = {
      files: [],
      fields: {}
    };
  });
});
```

Utworzyliśmy 2 pomocnicze zmienne:

- streams - tablica przechowująca obietnice które rozwiązują się kiedy strumień zapisu pliku przestaje płynąć

- parsingResult - obiekt który przechowuję strumień przesyłanego pliku oraz wszystkie inne pola

Przejdźmy dalej, busboy udostępnia nam możliwość nasłuchiwania na strumień z plikiem, ponieważ oprócz samego pliku busboy przekazuje nam też nazwe pliku, dzięki czemu możemy sprawdzić jakie rozszerzenie ma plik, i dla przykładu dopuścić tylko .jpg oraz .png.

```typescript
const supportedExtensions: string[] = [".png", ".jpg", ".jpeg"];

const isSupportedExtension = (filename: string): boolean =>
  supportedExtensions.includes(path.extname(filename));
```

Zróbmy prostą funkcje która pomaga nam utworzyć scieżke pod jaką zapiszemy plik

```typescript
const uploadPath: string = path.join(__dirname, "../upload");

const savePath = (uuid: string, filename: string): string => {
  return path.join(uploadPath, uuid + path.extname(filename));
};
```

```typescript
busboy.on("file", function(
  fieldname: string,
  file: NodeJS.ReadableStream,
  filename: string,
  encoding: string,
  mimeType: string
) {
  if (!isSupportedExtension(filename)) {
    reject(new Error("File extension is not supported"));
  } else {
    const uuid: string = uuidv1(); // uuid będzię nazwą pliku

    const parsedFile = {
      filename: filename,
      filePath: savePath(uuid, filename),
      encoding,
      fieldname,
      mimeType,
      file
    };

    parsingResult.files.push(parsedFile);

    const write = fs.createWriteStream(parsedFile.filePath);

    file.pipe(write); // podłączamy nasz strumień do strumienia z busboy

    streams.push(onStreamFinish(write)); // Do tablic streams wrzucamy Promise który wykona się dopiero jak zakończy się strumieniowanie naszego zapisu na dysk
  }

  file.resume(); // jeżeli wszystko działa do tej pory to włączamy strumień
});
```

Funkcja onStreamFinish jest funkcją która przyjmuje strumień i zwraca Promise która rozwiąże się dopiero wtedy kiedy cały strumień przestanie płynąć

```typescript
import { Stream } from "stream";

export function onStreamFinish(stream: Stream) {
  return new Promise((resolve, reject) => {
    stream.on("finish", resolve);
    stream.on("err", reject);
  });
}
```

Oprócz samych plików busboy parsuje również inne pola które możemy odebrać i zmapować do naszego obiektu parsingResult

```typescript
busboy.on("field", function(fieldname, value) {
  parsingResult.fields[fieldname] = value;
});
```

Ostatnim krokiem w pracy z busboy będzie podpięcie na zakończenie strumieniowania zwrócenia obietnicy

```typescript
busboy.on("finish", async () => {
  if (streams.length > 0) {
    Promise.all(streams)
      .then(resolve)
      .catch(reject);
  }
});

ctx.req.pipe(busboy);
```

Wszystkie do tej pory uzbierane strumienie zapisu, przeliczamy (tu możemy ustalić limit plików), i uruchamiamy. Kiedy wszystko rozwiążę się pomyślnie to rozwiązujemy naszą Promise zapisaną pod zmienną uploading. Kiedy już zarejestrowaliśmy działania dla strumieni możemy końcu przekierować nasz strumień z obiektu zapytania do busboy.

Kiedy już napisaliśmy kod odpowiedzialny za przyjęcie pliku przez busboy możemy poczekać na wykonanie naszej obietnicy i odesłać użytkownikowi odpowiednią informacje

```typescript
try {
  await uploading;

  ctx.body = { success: true };
} catch (error) {
  ctx.body = { success: false };
}
```

Tak oto zbudowana scieżka odbiera pliki od użytkownika.

## Podsumowanie

Praca z odbieraniem plików nie jest taka prosta, ale jeżeli tylko odpowiednio będziemy manewrować strumieniami to nie powinniśmy mieć żadnych problemów z zaimplementowaniem takiego uploadingu. Cały kod możesz znaleźć [tutaj](https://github.com/Rekseto/koa-file-uploading).
