# Por que este fork existe

Fork de [`tableflowhq/csv-import`](https://github.com/tableflowhq/csv-import) mantido pela Linkana
para remover duas vulnerabilidades altas que o pacote `csv-import-js` publicado no npm carrega.

## O problema

O `csv-import-js@1.0.18` do npm depende de `xlsx` (SheetJS CE) `^0.18.5`, afetado por:

| CVE | CVSS | Classe | Corrigido em |
| -- | -- | -- | -- |
| CVE-2023-30533 | 7.8 | Prototype Pollution | 0.19.3 |
| CVE-2024-22363 | 7.5 | ReDoS | 0.20.2 |

Nenhuma das duas versões corrigidas existe no npm: o pacote `xlsx` foi abandonado lá na 0.18.5 e o
SheetJS passou a distribuir só pelo CDN próprio, em `https://cdn.sheetjs.com/`. Por isso o Dependabot
reporta os dois achados como sem correção disponível.

Agrava o quadro o fato de o `index.js` publicado ser um bundle UMD que **embute** o SheetJS — não há
`require("xlsx")` nele. Um `override` do `xlsx` no projeto consumidor limparia o lockfile e calaria o
Dependabot sem trocar um byte do que roda no navegador.

## A correção

`xlsx` passa a ser instalado a partir do CDN do SheetJS, na 0.20.3, que cobre os dois CVEs:

```json
"xlsx": "https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz"
```

A API consumida em `src/importer/features/main/index.tsx` (`XLSX.read` e `XLSX.utils.sheet_to_json`)
é idêntica nas duas versões — a troca não exigiu mudança de código.

Junto vieram dois ajustes necessários para o build rodar em Node 24:

- **`@types/xlsx` removido.** É um stub deprecado que declara `xlsx: "*"` e reinstalaria a 0.18.5 do
  registry por baixo. O SheetJS 0.20.3 traz os próprios tipos.
- **`node-sass` removido.** Binário nativo deprecado que não compila em Node moderno. O `sass`
  (dart-sass) já estava nas devDependencies e é o fallback do `rollup-plugin-postcss`.
- **`packageManager` fixado em `yarn@3.6.4`**, que é o Yarn que gerou o `yarn.lock` do repositório.
  Sem isso o corepack cai no Yarn 1, que ignora o lockfile Berry e resolve tudo de novo.
- **`require("./package.json")` removido do `rollup.config-js.js`.** A variável não era usada e
  `require` não existe no config, que é ESM (`"type": "module"`).

## Como reconstruir o bundle

```bash
corepack yarn install
corepack yarn build:js
```

O artefato sai em `build/index.js` (UMD, global `CSVImporter`). É ele que a aplicação Rails da
Linkana versiona em `vendor/javascript/csv-import-js/`.

## Como conferir que o SheetJS antigo saiu

Comparando o bundle novo com o `csv-import-js@1.0.18` do npm: das 2.173 linhas exclusivas do
`xlsx@0.18.5`, 1.876 estavam no bundle do npm e **nenhuma** está no bundle deste fork.
