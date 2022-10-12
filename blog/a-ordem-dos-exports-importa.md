A ordem dos exports importa

<abbr title="muito grande;nem li">mg;nl</abbr>: quando você utiliza um arquivo index.js para exportar módulos no react, a ordem dos exports pode quebrar seu app se dois módulos importam um ao outro, causando um `Uncaught TypeError: _MyModule__WEBPACK_IMPORTED_MODULE_1__ is undefined`.

O react tem uma funcionalidade muito boa, o [absolute imports](https://create-react-app.dev/docs/importing-a-component/#absolute-imports), que ajuda você a não precisar ficar lidando com o caminho relativo entre os arquivos.

Se seu app tem uma estrutura do tipo

        - src/
            - components/
            - pages/

Você pode, num arquivo em `src/pages/Home.jsx`, importar módulos de duas formas

        // bad! >:(
        import { ButtonGroup } from '../components/ButtonGroup';
        import { MyButton } from '../components/RadioGroup';

        // good! >:)
        import { ButtonGroup, MyButton } from 'components';

Basta você largar um arquivo `jsconfig.json` na raíz do seu projeto, passando o diretório `src/` como `baseUrl`:

        {
          "compilerOptions": {
            "baseUrl": "src"
          },
          "include": ["src"]
        }

Porém existem algum gotchas quando você começa a agrupar seus exports.

<hr/>

## Agrupando exports

Pra que você possa agrupar todos os módulos em um mesmo import, você precisa utilizar um `index.js` no escopo de cada diretório.

    - src/
        - components/
            - index.js
            - ButtonGroup/
                - ButtonGroup.jsx
                - index.js
            - MyButton/
                - MyButton.jsx
                - index.js
        - pages/
            - ...

Para cada módulo, você precisa exportar no index.js o conteúdo do seu arquivo .jsx adjacente:

    // arquivo src/components/ButtonGroup/index.js

    export * from './ButtonGroup';

E no index do diretório em sí (components/, pages/, ..) você precisa exportar também cada módulo:

    // arquivo src/components/index.js

    // this is bad >:(

    export * from './ButtonGroup';
    export * from './MyButton';

Agora você pode acessá-los de fora.

## Gotcha nos exports

O que vai acontecer caso o ButtonGroup.jsx precise importar o componente MyButton?

Isso mesmo, um bug sinistro.

Abrindo o terminal você pode ver um `TypeError` não tratado:

        Uncaught TypeError: _MyButton__WEBPACK_IMPORTED_MODULE_1__ is undefined

A solução está na ordem dos exports: como o MyButton é acessado dentro do ButtonGroup, ele precisa ser exportado primeiro:

    // arquivo src/components/index.js

    // this is good >:)

    export * from './MyButton';
    export * from './ButtonGroup';

Bom desenvolvimento!

Tags: react, javascript
