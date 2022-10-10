Interceptando tokens com puppeteer

<abbr title="muito grande; nem li">mg;nl</abbr>: vamos ver como automatizar o login em um app e interceptar as requisições realizadas, com o objetivo de acessar os tokens gerados pelo front end de uma aplicação.

Digamos que você esteja trabalhando com uma API que exige autenticação.

Você consegue autenticar pelo frontend, mas gostaria de uma forma automatizada de acessar o token, para trabalhar
com as requisições pelo terminal, ou via postman/insomnia:

        curl -X POST -H 'Bearer tokensecretogeradonofrontend' https://api-autenticada.net/resources

Dependendo da solução de identidade e autenticação utilizada na API, você pode tentar reproduzir as requisições e gerar o token no seu formato final.

Mas alguns provedores, como o [Okta](https://www.okta.com/) ou o [Keycloak](https://www.keycloak.org/) dificultam essa tática, pois envolvem o uso de [nonces](https://pt.wikipedia.org/wiki/Nonce) ou processamento adicional dos valores trocados entre cliente e servidor.

Essas etapas são de difícil reprodução, e podem tomar muito tempo.

Uma alternativa é a automatização do acesso realizado no front end com um navegador `headless`, como o puppeteer.

<hr/>

## Identificando as requisições chave

Pra entender o funcionamento da API, e qual token você precisa acessar, recomendo utilizar a aplicação com o monitor de rede aberto.

Geralmente, existe um (ou vários) endpoints que envolvem a troca de um par de login/senha por um valor intermediário, que depois é utilizado para produzir um token.

Esse token vai ser utilizado como cabeçalho para as futuras requisições.

## Automatizando a extração do token com puppeteer

No exemplo abaixo, tenho uma aplicação com o frontend rodando em <http://localhost:3000>, que por sua vez acessa uma API em <http://localhost:8080>, autenticada através do Okta.

A requisição que retorna o token pro frontend foi identificada como sendo um `POST` para <http://localhost:8080/api/token>

O puppeteer é configurado para acessar o frontend, inserir valores num formulário de login e apertar o botão de enviar o formulário.

Dali em diante, adicionamos um `listener` em todas as requisições finalizadas (`requestfinished`), e filtramos a requisição que trás como resposta o token, com a função `onRequestFinished`.

Coloquei uma verificação adicional (`request.method()` precisa ser `POST`), pois é comum o envio de uma requisição `OPTIONS` pelo frontend antes da requisição verdadeira.


        // get_token.js
        const puppeteer = require("puppeteer");

        const onRequestFinished = async (request) => {
          if (request.url() !== 'http://localhost:8080/api/token') {
            return;
          }
          if (request.method() !== 'POST') {
            return;
          }
          const response = request.response();
          try {
            const responseContent = await response.json();
            console.log(JSON.stringify(responseContent));
          } catch (e) {
            console.log(`Could not parse JSON response, details: ${e}`);
          }
          return;
        }

        (async () => {
          const browser = await puppeteer.launch({ headless: true });
          const page = await browser.newPage();
          await page.goto("http://localhost:3000", { waitUntil: "load" });

          await page.waitForSelector("input[type=email]", {
            visible: true,
          });
          await page.focus("input[type=email]");
          await page.type("input[type=email]", "john@doe.com");

          await page.focus("input[type=password]");
          await page.type("input[type=password]", "mystrongpassword");

          await page.click("button.btn-primary");

          page.on("requestfinished", onRequestFinished);
          await page.waitForNavigation({ waitUntil: "networkidle0" });
          await browser.close();
        })();

No exemplo, assumi que o retorno da requisição é no formato json.

Rodando o script com

        node get_token.js | jq -r .

Temos um output no formato:

        {
          "token": "eyJhbGciOiJlUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6NSwiZW1haWwiOiJqb2FvQGV4YW1wbGUuY29tIiwicm9sZSI6W10sImlhdCI6MTY2NTQwODg4OCwiZXhwIjoxNjY1NDM3Njg4LCJhdWQiOiJlcHNvbiIsImlzcyI6ImVwc29uIiwic3ViIjoiam9hb0BleGFtcGxlLmNvbSJ9.Q6PBDCWjvguo123Scb1TZ3SuhQ8Mw_CAx4R5Q-KciY",
          "name": "JOHN DOE",
          "registration": null,
          "rfidCode": null,
          "birthDate": null,
          "admissionDate": null,
          "demissionDate": null,
          "status": true,
          "createdBy": null,
          "updatedBy": null,
          "createdAt": "2022-10-05T13:30:07.000Z",
          "updatedAt": "2022-10-05T13:30:07.000Z",
          "deletedAt": null,
          "teamId": null,
          "roleId": null,
          "team": null
        }

E a requisição final fica no formato:

        curl -X POST \
        -H 'Bearer eyJhbGciOiJlUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6NSwiZW1haWwiOiJqb2FvQGV4YW1wbGUuY29tIiwicm9sZSI6W10sImlhdCI6MTY2NTQwODg4OCwiZXhwIjoxNjY1NDM3Njg4LCJhdWQiOiJlcHNvbiIsImlzcyI6ImVwc29uIiwic3ViIjoiam9hb0BleGFtcGxlLmNvbSJ9.Q6PBDCWjvguo123Scb1TZ3SuhQ8Mw_CAx4R5Q-KciY' \
        http://localhost:8000/resources


Tags: node, puppeteer, oauth2
