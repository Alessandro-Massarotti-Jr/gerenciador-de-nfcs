---
name: web-nfc
description: Guia para usar a Web NFC API do navegador (NDEFReader, NDEFMessage, NDEFRecord) para ler e gravar tags NFC em formato NDEF. Use sempre que for implementar, revisar ou depurar código que leia/escreva tags NFC no navegador, trate permissões "nfc", decodifique registros NDEF (text, url, mime, smart-poster, external) ou lide com erros como NotAllowedError/NotSupportedError do NDEFReader.
---

# Web NFC API

Fonte: MDN — https://developer.mozilla.org/en-US/docs/Web/API/Web_NFC_API
Especificação: https://w3c-cg.github.io/web-nfc/

A Web NFC API troca dados com tags NFC usando mensagens **NDEF** (NFC Data Exchange Format). É **experimental** e **não é Baseline**: na prática funciona só no **Chrome para Android** (versão 89+). Não há suporte em desktop, iOS/Safari ou Firefox.

## Restrições que valem para todo código

- **Só em contexto seguro (HTTPS)**, ou `localhost` em desenvolvimento.
- **Só no frame de nível superior e com a aba visível.** Leituras param quando a aba vai para segundo plano.
- **`scan()` e `write()` precisam de gesto do usuário** (clique ou toque) na primeira vez, porque abrem o prompt da permissão `"nfc"`.
- **Só NDEF.** A tag precisa estar formatada em NDEF. Não há operações de baixo nível (APDU, MIFARE raw e similares).
- Sempre faça detecção de recurso antes de usar:

```js
if (!("NDEFReader" in window)) {
  // Mostre uma mensagem: "NFC não suportado neste navegador/dispositivo"
}
```

- Verificar a permissão sem abrir o prompt (útil para iniciar o scan automaticamente se a permissão já foi concedida):

```js
const { state } = await navigator.permissions.query({ name: "nfc" });
// "granted" | "prompt" | "denied"
```

## Interfaces

| Interface | Papel |
|---|---|
| `NDEFReader` | Ponto de entrada. Lê (`scan`) e grava (`write`) tags. É um `EventTarget`. |
| `NDEFMessage` | Mensagem lida ou a gravar. Propriedade `records` (somente leitura): array de `NDEFRecord`. |
| `NDEFRecord` | Um registro da mensagem: `recordType`, `mediaType`, `id`, `data`, `encoding`, `lang`, mais o método `toRecords()`. |
| `NDEFReadingEvent` | Evento `reading`: `serialNumber` (string, ou `""` se indisponível) e `message` (`NDEFMessage`). |

### NDEFReader

```js
const ndef = new NDEFReader();
```

**`scan(options?) → Promise<void>`**
- Ativa a leitura. Resolve assim que a leitura é agendada, **não** quando uma tag é lida. As tags chegam pelo evento `reading`.
- `options.signal`: `AbortSignal` para cancelar o scan.
- Rejeita com `DOMException`:
  - `AbortError`: abortado pelo signal.
  - `InvalidStateError`: já existe um scan em andamento.
  - `NotAllowedError`: permissão negada.
  - `NotSupportedError`: não há adaptador NFC compatível, ou não foi possível conectar.

**`write(message, options?) → Promise<void>`**
- Grava na próxima tag aproximada. Resolve quando a gravação termina.
- `message` pode ser:
  - uma `string` (vira um registro `text`);
  - um `ArrayBuffer`, `TypedArray` ou `DataView` (vira um registro `mime` com `application/octet-stream`);
  - um objeto `{ records: [...] }` (veja "Init de registro" abaixo).
- `options.overwrite` (boolean, padrão `true`): se for `false`, rejeita quando a tag já tiver dados.
- `options.signal`: `AbortSignal` para cancelar a gravação.
- Rejeita com `DOMException`:
  - `AbortError`: abortado pelo signal.
  - `NotAllowedError`: permissão negada, ou `overwrite: false` com a tag já preenchida.
  - `NotSupportedError`: sem adaptador NFC, o adaptador não grava, ou não foi possível conectar.
  - `NotReadableError`: o navegador não pode acessar o adaptador (por exemplo, NFC desligado nas configurações).
  - `NetworkError`: a transferência falhou no meio, por exemplo porque a tag foi afastada.

**`makeReadOnly(options?) → Promise<void>`**
- Está na especificação e no Chrome 100+, mas não tem página no MDN.
- Torna a tag **permanentemente somente leitura** (irreversível). Aceita `options.signal`.
- Peça confirmação explícita ao usuário antes de chamar.

**Eventos**
- `reading` (`NDEFReadingEvent`): uma tag NDEF foi lida.
- `readingerror`: há uma tag no campo, mas ela não pôde ser lida (não é NDEF, está corrompida etc.).

### Init de registro (para `write` e `new NDEFRecord(init)`)

| Campo | Obrig. | Descrição |
|---|---|---|
| `recordType` | sim | Tipo do registro (veja a tabela abaixo). |
| `data` | não | `string`, `ArrayBuffer`, `TypedArray`, `DataView` ou array de registros aninhados. |
| `mediaType` | não | MIME type. Usado com `recordType: "mime"`. |
| `id` | não | Identificador definido pelo desenvolvedor (URL absoluta ou relativa). |
| `encoding` | não | Encoding do texto (`"utf-8"` é o padrão). |
| `lang` | não | Tag de idioma BCP 47 (ex.: `"pt-BR"`). |

### Valores de `recordType`

| Valor | Uso | Como ler `data` |
|---|---|---|
| `"empty"` | Registro vazio | — |
| `"text"` | Texto (NDEF-TEXT). Aceita `encoding` e `lang`. | `new TextDecoder(record.encoding).decode(record.data)` |
| `"url"` | URI (NDEF-URI). Escreva em minúsculas: a página do MDN lista `"URL"`, mas o valor da especificação e do Chrome é `"url"`. | `new TextDecoder().decode(record.data)` |
| `"absolute-url"` | URL absoluta | `new TextDecoder().decode(record.data)` |
| `"mime"` | Dados binários ou JSON com `mediaType` | Depende do `mediaType`. Para JSON: `JSON.parse(new TextDecoder().decode(record.data))` |
| `"smart-poster"` | Smart poster (URL, título, ícone e ação) | `record.toRecords()` |
| `"unknown"` | Tipo desconhecido | Bytes brutos |
| `"dominio.com:tipo"` | **Tipo externo** (domínio + `:` + nome) | Bytes, ou `record.toRecords()` se o registro tiver registros aninhados |
| `":tipo"` | **Tipo local**, válido só aninhado em outro registro | Igual ao tipo externo |

`record.data` é sempre um `DataView` (ou `null`). `toRecords()` converte o payload em registros aninhados. Use-o para `smart-poster` e para tipos externos ou locais.

## Padrões de uso

### 1. Ler tags

```js
const ndef = new NDEFReader();
const controller = new AbortController();

await ndef.scan({ signal: controller.signal });

ndef.addEventListener("readingerror", () => {
  console.warn("Tag detectada, mas não foi possível ler. Ela está em NDEF?");
});

ndef.addEventListener("reading", ({ serialNumber, message }) => {
  console.log("Serial:", serialNumber);
  for (const record of message.records) {
    console.log(record.recordType, record.mediaType, decodeRecord(record));
  }
});

// Para parar de ler: controller.abort();
```

### 2. Decodificar registros

```js
function decodeRecord(record) {
  switch (record.recordType) {
    case "empty":
      return null;
    case "text":
      return new TextDecoder(record.encoding ?? "utf-8").decode(record.data);
    case "url":
    case "absolute-url":
      return new TextDecoder().decode(record.data);
    case "mime":
      if (record.mediaType === "application/json") {
        return JSON.parse(new TextDecoder().decode(record.data));
      }
      if (record.mediaType?.startsWith("text/")) {
        return new TextDecoder().decode(record.data);
      }
      return record.data; // DataView: binário, imagem etc.
    case "smart-poster":
      return record.toRecords().map(decodeRecord);
    default:
      // Tipo externo ("dominio:tipo"), local (":tipo") ou "unknown"
      try {
        return record.toRecords().map(decodeRecord);
      } catch {
        return record.data;
      }
  }
}
```

### 3. Gravar

```js
const ndef = new NDEFReader();

// Texto simples
await ndef.write("Olá mundo");

// Vários registros
await ndef.write({
  records: [
    { recordType: "text", data: "Sala 12", lang: "pt-BR" },
    { recordType: "url", data: "https://exemplo.com/sala/12" },
    {
      recordType: "mime",
      mediaType: "application/json",
      data: new TextEncoder().encode(JSON.stringify({ id: 12, andar: 2 })),
    },
  ],
});

// Não sobrescrever uma tag que já tem dados
await ndef.write("novo", { overwrite: false }); // NotAllowedError se a tag já tiver dados

// Tipo externo com registros aninhados
await ndef.write({
  records: [{
    recordType: "exemplo.com:inventario",
    data: { records: [{ recordType: "text", data: "item-42" }] },
  }],
});
```

### 4. Gravar com timeout

```js
async function writeWithTimeout(data, ms = 5000) {
  const ndef = new NDEFReader();
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), ms);
  try {
    await ndef.write(data, { signal: controller.signal });
  } finally {
    clearTimeout(timer);
  }
}
```

### 5. Gravar sem disparar a leitura

Quando há um scan ativo, aproximar a tag para gravar também dispara `reading`. Ignore essa leitura enquanto a gravação está pendente. Gravar a partir do evento `reading` também permite validar a tag antes, por exemplo pelo `serialNumber`.

```js
const ndef = new NDEFReader();
let ignoreRead = false;

ndef.onreading = (event) => {
  if (ignoreRead) return; // há uma gravação pendente
  // ...processar a leitura normal
};

function write(data) {
  ignoreRead = true;
  return new Promise((resolve, reject) => {
    ndef.addEventListener("reading", (event) => {
      // Valide event.serialNumber aqui, se precisar
      ndef.write(data).then(resolve, reject).finally(() => (ignoreRead = false));
    }, { once: true });
  });
}

await ndef.scan();
await write("Olá mundo");
```

### 6. Tratar erros

```js
function nfcErrorMessage(err) {
  switch (err?.name) {
    case "NotAllowedError":   return "Permissão de NFC negada, ou a tag já tem dados (overwrite: false).";
    case "NotSupportedError": return "Este dispositivo não tem NFC compatível com Web NFC.";
    case "NotReadableError":  return "NFC indisponível. Verifique se o NFC está ligado nas configurações.";
    case "NetworkError":      return "A gravação foi interrompida. Mantenha a tag encostada até o fim.";
    case "InvalidStateError": return "Já existe uma leitura em andamento.";
    case "AbortError":        return "Operação cancelada ou tempo esgotado.";
    default:                  return `Erro de NFC: ${err?.message ?? err}`;
  }
}
```

## Checklist ao implementar

1. Detectar `"NDEFReader" in window` e ter uma interface alternativa (por exemplo, entrada manual ou QR code) quando não houver suporte.
2. Chamar `scan()` ou `write()` dentro de um handler de clique ou toque.
3. Usar uma única instância de `NDEFReader` para o scan. Um segundo `scan()` em andamento gera `InvalidStateError`.
4. Usar `AbortController` para parar o scan ao desmontar o componente ou sair da tela. Em React, faça isso no cleanup do `useEffect`.
5. Tratar `readingerror` com uma mensagem amigável.
6. Decodificar `data` conforme o `recordType` e o `mediaType`. Nunca presuma que é texto.
7. Usar `"url"` em minúsculas no `recordType`.
8. Pedir confirmação antes de `overwrite` em tag já preenchida e, principalmente, antes de `makeReadOnly()`, que é irreversível.
9. Tipagem TypeScript: os tipos não vêm no `lib.dom`. Instale `@types/w3c-web-nfc` e adicione `"w3c-web-nfc"` em `compilerOptions.types`.
10. Testar em um aparelho Android real com Chrome via HTTPS. Para depurar, use `chrome://inspect` com depuração USB.
