# Gerenciador de NFCs

<p>
  <img src="https://img.shields.io/badge/made%20by-Alessandro%20Massarotti%20Jr-1c1c1c?style=flat-square&labelColor=5f5f5d">
  <img src="https://img.shields.io/badge/HTML5-1c1c1c?style=flat-square&logo=html5&logoColor=fcfbf8">
  <img src="https://img.shields.io/badge/CSS3-1c1c1c?style=flat-square&logo=css&logoColor=fcfbf8">
  <img src="https://img.shields.io/badge/JavaScript-1c1c1c?style=flat-square&logo=javascript&logoColor=fcfbf8">
  <img src="https://img.shields.io/badge/Web%20NFC-Chrome%20Android%2089+-1c1c1c?style=flat-square&labelColor=5f5f5d&logo=googlechrome&logoColor=fcfbf8">
  <img alt="GitHub language count" src="https://img.shields.io/github/languages/count/Alessandro-Massarotti-Jr/gerenciador-de-nfcs?color=1c1c1c&labelColor=5f5f5d&style=flat-square">
  <img alt="GitHub Top Language" src="https://img.shields.io/github/languages/top/Alessandro-Massarotti-Jr/gerenciador-de-nfcs?color=1c1c1c&labelColor=5f5f5d&style=flat-square">
</p>

Página web para ler, gravar e gerenciar tags NFC direto do navegador do celular, usando a [Web NFC API](https://developer.mozilla.org/en-US/docs/Web/API/Web_NFC_API). Não precisa instalar app, framework nem fazer build: é um único `index.html`.

## Funcionalidades

### Ler

- Mostra o número de série (UID) da tag
- Lista e decodifica cada registro NDEF: texto, link, e-mail, telefone, SMS, localização, contato (vCard), Wi-Fi, Bluetooth, JSON, smart poster e tipos externos
- Copia o valor de um registro para a área de transferência
- Lista as leituras da sessão atual
- **Editar e regravar:** leva os registros da tag lida para a aba Gravar

### Gravar

- Monta uma lista com vários registros e grava todos de uma vez
- Os registros podem ser reordenados
- Tipos disponíveis:

| Tipo | Formato NDEF |
|---|---|
| Texto | `text` |
| Link | `url` |
| E-mail | `url` (`mailto:`) |
| Telefone | `url` (`tel:`) |
| SMS | `url` (`sms:`) |
| Localização | `url` (`geo:`) |
| Contato e contato de emergência | `mime` (`text/vcard`) |
| Wi-Fi | `mime` (`application/vnd.wfa.wsc`) |
| Bluetooth | `mime` (`application/vnd.bluetooth.ep.oob`) |
| JSON | `mime` (`application/json`) |

- Estima o espaço que a mensagem vai ocupar em cada chip: Ultralight EV1, NTAG213, NTAG215, NTAG216 e Mifare Classic 1K
- Tem a opção de não sobrescrever uma tag que já tenha dados

### Outros

- **Copiar tag:** lê uma tag de origem e grava o mesmo conteúdo numa tag de destino
- **Apagar:** substitui o conteúdo por um registro vazio
- **Bloquear permanentemente:** deixa a tag somente leitura. É irreversível, por isso a página pede que você digite `BLOQUEAR` para confirmar. Exige Chrome 100 ou mais recente.

## Requisitos

- **Chrome para Android 89 ou mais recente**, num celular com NFC ligado. Outros navegadores, Chrome no computador e iOS não têm Web NFC.
- **Página servida por HTTPS**, ou por `localhost` durante o desenvolvimento.
- **Tags formatadas em NDEF.** Tags novas costumam vir assim de fábrica.

## Como executar

Não há dependências para instalar.

### Publicar com HTTPS

Publique o `index.html` em qualquer hospedagem estática com HTTPS (GitHub Pages, Netlify, Vercel…) e abra o endereço no Chrome do celular.

### Testar localmente pelo celular

1. Suba um servidor estático na pasta do projeto:

   ```bash
   npx serve .
   ```

2. Conecte o celular ao computador por USB, com a depuração USB ativada.
3. No Chrome do computador, abra `chrome://inspect/#devices`.
4. Em **Port forwarding**, encaminhe a porta do servidor (por exemplo, `3000` → `localhost:3000`).
5. No Chrome do celular, abra `http://localhost:3000`.

O próprio `chrome://inspect` também abre o console da página do celular para depuração.

## Estrutura

```
.
├── index.html              # aplicação completa (HTML, CSS e JS)
├── DESIGN.md               # design system usado no layout
└── .claude/skills/web-nfc  # guia da Web NFC API
```

<br>

---

Desenvolvido Por [Alessandro Massarotti Jr](https://github.com/Alessandro-Massarotti-Jr) 🤖
