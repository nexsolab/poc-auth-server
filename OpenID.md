# Servidor de autenticação

Este documento apresenta minhas anotações sobre autenticação de usuários para criar um servidor provedor de autenticação baseado no OpenID e o protocolo OAuth 2.0 em [node.js](nodejs.org).  
Inspirado em como as grandes empresas, como Microsoft, Google, Facebook, Twitter realizam a autenticação de usuários e permitem que terceiros interajam com uma aplicação de forma autenticada.

## Porque Node.js?

Eu tinha somente a opção de escolher o uso das linguagens C# e JavaScript, com preferência para esse último.  
A biblioteca do JavaScript se mostrou muito melhor em termos de documentação, sem estar atrelada à telas prontas e ainda é o pacote mais certificado e com mais opções de uso, conforme abaixo.

### Diferença das implementações em C# e Node.js

|Conjunto|Perfil|Node.js (node oidc-provider)|C# (IdentityServer4)|
|---|---|---|---|
|[Servers](https://openid.net/developers/certified/#RPServices)|Basic OP|✅|✅|
|[Servers](https://openid.net/developers/certified/#RPServices)|Implicit OP|✅|✅|
|[Servers](https://openid.net/developers/certified/#RPServices)|Hybrid OP|✅|✅|
|[Servers](https://openid.net/developers/certified/#RPServices)|Config OP|✅|✅|
|[Servers](https://openid.net/developers/certified/#RPServices)|Dynamic OP|✅||
|[Servers](https://openid.net/developers/certified/#RPServices)|Form Post OP|✅||
|[Servers](https://openid.net/developers/certified/#RPServices)|3rd Party-Init OP|✅||
|[Logout](https://openid.net/developers/certified/#OPLogout)|RP-Initiated OP|✅||
|[Logout](https://openid.net/developers/certified/#OPLogout)|Session OP|✅||
|[Logout](https://openid.net/developers/certified/#OPLogout)|Front-Channel OP|✅||
|[Logout](https://openid.net/developers/certified/#OPLogout)|Back-Channel OP|✅||
|[FAPI](https://openid.net/developers/certified/#FAPIOP)|FAPI R/W OP w/ MTLS|✅||
|[FAPI](https://openid.net/developers/certified/#FAPIOP)|FAPI R/W OP w/ Private Key|✅||

Todas as outras linguagens, frameworks, serviços online e na nuvem não oferecem tantos perfis quanto o `node oidc-provider` que é a implementação mais completa.
Além disso, ele oferece somente a especificação, sem estar preso à um framework ou telas prontas.

## Sobre este documento

Este documento não é um guia oficial e possui uma linguagem mais direta e informal dos conceitos. É puramente anotações e interpretações minhas sobre a leitura de toda documentação e experiências profissionais.  
Além disso esse documento é um estudo para um projeto pessoal, e podem fazer referências à tecnologias utilizadas pelo projeto, não necessárias para o servidor de autenticação ou para entender o conceito.  

É focado principalmente na criação de um servidor de autenticação, porém as anotações são para interação com o servidor, não entrando em especificações de como um servidor OP deva funcionar, tornando por base que o `node oidc-provider` já tem em sua implementação as validações e regras no lado do servidor.

> OBS: muitas vezes é usado a sigla **IdP** para referir-se ao servidor de autenticação (Authorization Server, Identity Provider ou IdP, OpenID Provider ou OP), ou seja, o lugar onde está sendo executado o node oidc-provider.

### Leia a documentação Oficial

- [RFC6749 - OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
- [RFC6750 - Bearer Token Usage](https://tools.ietf.org/html/rfc6750)
- [OpenID Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [node oidc-provider](https://github.com/panva/node-oidc-provider)
- [node openid-client](https://www.npmjs.com/package/openid-client)
- [RFC7515 - JWS](https://tools.ietf.org/html/draft-ietf-jose-json-web-signature-41)

#### Requisitos para entendimento

- Protocolo HTTP
- OAuth 2.0 (recomendável)
- JWT (Header, Payload, Signature)

## Conceitos

- **Identity** (ou Identidade) é um conjunto de atributos relacionados à uma entidade (ISO29115). No mundo real a entidade *pessoa* pode ter várias identidades: Pai, Funcionário, Marido...  
- Uma camada de Identidade provê (5W1H):
  - **Who:** quem é o usuário autenticado
  - **Where:** onde ele está autenticado
  - **When:** quando ele se autenticou
  - **How:** como se autenticou
  - **What:** quais atributos ele pode oferecer
  - **Why:** porquê ele está oferecendo
- OpenID = **autenticação** (certifica que você é você)
- OAuth 2.0 = **autorização** (certifica que você acesse só o que você tem permissão)

## OpenID Connect

As anotações abaixo referenciam a especificação OpenID Connect Core 1.0.  
Esta é base principal do OpenID, dos fluxos de autenticação.

### Fluxos de autenticação (_flows_)

#### 1. Authorization Code

O fluxo mais comum. Uma aplicação percebe que não há sessão e redireciona o usuário para uma tela de login. Após informado usuário e senha, este redireciona para a aplicação de volta junto com um código temporário. No back-end, o código temporário serve para trocar o código por um token de acesso e de id.

Vantagens:

- Não expõe o token para o navegador (User Agent).  
- O cliente recebe publicamente um código, e internamente troca por um token de acesso.

##### Requisição da Autenticação

```http
HTTP/1.1 302 Found
Location: https://server.example.com/authorize?
    response_type=code
    &scope=openid%20profile%20email
    &client_id=s6BhdRkqt3
    &state=af0ifjsldkj
    &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
```

|Parâmetro|Valor e descrição|
|---|---|
|`response_type`|`code`|
|`scope`|`openid`|
|`client_id`|_Código da aplicação requisitando um token._|
|`redirect_uri`|_Deve ser `https`._<br>_Deve estar registrado no IdP._|
|`state`|_Evita CSRF e XSRF._<br>_Pode ser o hash de um cookie do navegador no cliente._|
|`nonce`|_Deve ser um hash relacionado à sessão (pode ser baseado em cookie HttpOnly com um valor aleatório._|
|`prompt`|_Lista separada por espaço de como o IdP pergunta pro usuário sobre consentimento ou re-autenticação._<br>`none` não faz nada (usado pra checar se tem permissões e sessão)<br>`login` (só mostra tela de autenticação)<br>`consent` (só pede consentimento)<br>`select_account` (pede para o usuário escolher uma das contas ativas)|
|`max_age`|Tempo máximo em segundos em que um usuário é considerado autenticado.<br>Quando esse tempo termina, uma requisição nova no endpoint de autorização, irá pedir que o usuário se autentique novamente).|
|`ui_locales`|Lista separada por espaço e em ordem de preferência dos idiomas para a tela de login. Ex: `pt-BR pt en`
|`id_token_hint`|Envia um token previamente emitido pelo IdP. Retorna sucesso quando já está autenticado, ou um erro caso contrário.<br>Deve ser usado quando o `prompt=none` é informado.<br>O próprio IdP tem que estar como audiência `aud` do token.|
|`login_hint`|Informa um endereço de `e-mail` ou `telefone` que será preenchido automaticamente na tela de login.|

##### Erros de Autenticação

```http
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
    error=invalid_request
    &error_description=Unsupported%20response_type%20value
    &state=af0ifjsldkj
```

|Erro|Descrição|
|---|---|
|`interaction_required`|Quando `prompt=true` porém é preciso exibir uma tela.|
|`login_required`|Quando `prompt=true` porém é preciso exibir a tela de login.|
|`account_selection_required`|Quando `prompt=true` porém é preciso exibir a tela para selecionar a sessão.|
|`consent_required`|Quando `prompt=true` porém é preciso exibir a tela de consentimento.|
|`invalid_request_uri`|Quando `request_uri` está inválido.|
|`invalid_request_object`|Algo na requisição é inválido.|
|`request_not_supported`|Não suporta a requisição informada.|
|`request_uri_not_supported`|Não suporta o `request_uri` informado.|
|`registration_not_supported`|Não suporta o `registration` informado.|

##### Validação da resposta da Autenticação

```http
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
    code=SplxlOBeZQQYbYS6WxSbIA
    &state=af0ifjsldkj
```

|Item|Validação|
|---|---|
|`code`|Obrigatório.|
|`state`|Obrigatório se foi informado na requisição.<br>Deve ser idêntico ao informado antes.|

##### Requisição de Token

```http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
    &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
```

|Parâmetro|Valor e descrição|
|---|---|
|`grant_type`|`authorization_code`|
|`code`|O código recebido na requisição de Autenticação.|
|`redirect_uri`|A URL que será redirecionado após o login.<br>Note que essa URL precisa ser idêntica a informada na requisição de Autenticação.|
|`client_id`|O identificador do cliente (aplicação).|

##### Erro na requisição do Token

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
    "error": "invalid_request"
}
```

##### Validação da resposta do Token

```http
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"example",
    "expires_in":3600,
    "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
    "example_parameter":"example_value"
}
```

|Item|Validação|
|---|---|
|`access_token`|Deve existir.|
|`token_type`|Deve ser `Bearer`.|
|`id_token`|Deve existir e validar conforme [descrito abaixo](#validação-do-id-token).|
|`expires_in`|Segundos em que o token irá expirar após a requisição.|
|`refresh_token`|Token que pode ser usado para renovar o atual após a expiração.|
|`scope`|Deve existir se foi informado na requisição, e deverá ser idêntico.|

#### 2. Implícito

- Retorna o token de acesso em um única requisição (o endpoint de Token não é utilizado), útil quando trabalhamos com SPAs, por exemplo.
- Em contra partida, tem uma segurança menor e por segurança **não** deve ser usado em aplicações que não são SPA (como as que tem um back-end).

##### Requisição implícita

A requisição de autenticação implícita, acontece da mesma forma que a [requisição de autorização por código de autorização](#requisição-da-autenticação), exceto por alguns parâmetros diferentes:

|Parâmetro|Valor e descrição|
|---|---|
|`response_type`|Deve ser `id_token token`.|
|`nonce`|Se torna obrigatório ser informado.|

A validação e outros processos ocorrem da mesma forma que a [requisição de autorização por código de autorização](#requisição-da-autenticação), porém ao obter uma resposta, no redirecionamento de volta para a aplicação é enviados todos os parâmetros direto na URL:

|Parâmetro|Descrição|
|---|---|
|`access_token`|O token de acesso.|
|`token_type`|O tipo do token obtido. Sempre `Bearer`.|
|`id_token`|O token de ID.|
|`state`|O valor de estado informado na requisição.|
|`expires_in`|Segundos em que o token irá expirar.|

Por exemplo:

```http
HTTP/1.1 302 Found
Location: https://client.example.org/cb#
  access_token=SlAV32hkKG
  &token_type=bearer
  &id_token=eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso
  &expires_in=3600
  &state=af0ifjsldkj
```

Quando receber a resposta, é interessante a aplicação validar em um servidor (back-end) o que foi recebido. Um exemplo de código JS presente na página de callback pode ser:

```html
<script type="text/javascript">
// Primeiro pegamos os parâmetros
var params = {},
    postBody = location.hash.substring(1),
    regex = /([^&=]+)=([^&]*)/g,
    m;
while (m = regex.exec(postBody)) {
    params[decodeURIComponent(m[1])] = decodeURIComponent(m[2]);
}

// E enviamos o token para os servidor
var req = new XMLHttpRequest();
// usando o POST, assim não é registrado no histórico do navegador
req.open('POST', 'https://' + window.location.host + '/catch_response', true);
req.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');

req.onreadystatechange = function (e) {
    if (req.readyState === 4) {
        if (req.status === 200) {
            // Se foi OK, então redireciona para a tela principal
            window.location = 'https://' + window.location.host + '/apos_login'
        }
        // Se for inválido, gera um erro
        else if (req.status === 400) {
            alert('Ocorreu um erro ao processar o token.')
        } else {
            alert('Alguma coisa deu errada.')
        }
    }
};
req.send(postBody);
</script>
```

##### Token da autenticação implícita

Esses parâmetros adicionais estarão presentes no ID Token recebido:

|Claim|Descrição|
|---|---|
|`nonce`|O número aleatório informado antes.|
|`at_hash`|Um hash que representa o token recebido e deve ser usado para garantir a legitimidade do mesmo.|

#### 3. Autenticação híbrida

A autenticação híbrida é uma mistura do implícito e do código de autorização, recebendo assim alguns tokens direto na URL de callback e outros através da [requisição de Token](#requisição-de-token).

A requisição, assim como outros processos, também acontece da mesma forma que a [requisição de autorização por código de autorização](#requisição-da-autenticação), exceto que:

|Parâmetro|Valor e descrição|
|---|---|
|`response_type`|Deve especificar o que será retornado diretamente, como:<br>`code id_token`, `code token`, `code id_token token`.|

O tipo escolhido de resposta indicará o que irá retornar direto na URL de callback:

|Tipo de resposta|Descrição|
|---|---|
|`code`|Obrigatório. Irá sempre retornar o código de autorização.|
|`id_token`|Retorna o token de ID para identificação do usuário.|
|`token`|Retorna o token de acesso.|

Os tokens que não são retornados diretamente, podem ser obtidos usando o code na [requisição de token](#requisição-de-token).

#### 4. Fluxo do dispositivo [RFC 8628](https://tools.ietf.org/html/rfc8628)

Este fluxo acontece quando temos dispositivos conectados à Internet que não possuem um navegador ou não tem uma forma de escrever dados (como a falta de um teclado, por exemplo). Ele permite que os clientes OAuth presentes nestes dispositivos (como Smart TVs, quadros digitais e impressoras) possam obter um token de acesso.

Para que isso ocorra, é importante assegurar que:

- Os dispositivo está conectado à Internet e pode fazer chamadas HTTPS de saída.
- O dispositivo é capaz de exibir ou informar uma URI e um código para o usuário
- O usuário tem um dispositivo secundário (como um PC ou smartphone) que possa interagir com a requisição.

Após mostrar o código para o usuário, o dispositivo fica fazendo chamadas repetidas para o servidor de autorização até que ele obtenha acesso, se não houver uma forma de enviar requisições de entrada para o dispositivo.

Normalmente o dispositivo só interage com um único servidor de autorização.

##### 1. Autenticação do dispositivo

###### 1.1 Requisição de códigos

A primeira etapa, é o dispositivo solicitar os códigos de verificação para o servidor de autorização no endpoint de autenticação de dispositivos, informando o ID do cliente (`client_id`) e escopos (`scope`) desejados.

```http
POST /device_authorization HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded

client_id=1406020730&scope=example_scope
```

> Todas as requisições devem ser feitas através de TLS/SSL.

###### 1.2 Resposta de códigos

O servidor de autorização responde com um JSON informando dois códigos de verificação: de dispositivo e de usuário que valem por determinado tempo.

|Campo|Descrição|
|---|---|
|`device_code`|O código de verificação do dispositivo.|
|`user_code`|O código de verificação do usuário.|
|`verification_uri`|A URI que deve ser curta e será exibida para o usuário digitar em outro dispositivo.|
|`verification_uri_complete`|Uma URI opcional que pode incluir o código de usuário (ou funcionalidade similar).|
|`expires_in`|O tempo em segundos em que os códigos irão expirar.|
|`interval`|O tempo em segundos em que o dispositivo deve aguardar antes de fazer uma nova requisição para o IdP, perguntando sobre o estado atual.<br>Por padrão esse valor é `5`.|

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "device_code": "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
  "user_code": "WDJB-MJHT",
  "verification_uri": "https://example.com/device",
  "verification_uri_complete":
      "https://example.com/device?user_code=WDJB-MJHT",
  "expires_in": 1800,
  "interval": 5
}
```

##### 2. Interação do usuário

Quando o dispositivo recebe os códigos de verificação, ele deverá mostrar a URI de autorização (`verification_uri`) e o código de usuário (`user_code`) para o usuário, solicitando-o que acessa a URI em um outro dispositivo e para que digite o código informado.

> Note que o código de verificação de dispositivo (`device_code`) não deve ser exibido!

Quando o campo `verification_uri_complete` é recebido, o dispositivo pode exibi-lo de uma forma não textual (devido a URI longa e já informando o código de usuário), como por exemplo, exibindo um QR Code que irá abrir essa URL ou através de NFC. Porém é recomendável que o `verification_uri` continue sendo exibido.

No servidor de autorização, deve fornecer uma tela para que seja informado o código de usuário. O servidor pode requisitar outros fluxos e processos antes, como solicitar a autenticação do usuário ou seu cadastro, solicitar um código OTP, entre outros.

Após informar o código, ou após acessar a URI completa (via QR Code por exemplo), o servidor deverá exibir uma tela solicitando que o usuário aprove ou rejeite o uso daquele dispositivo.

##### 3. O papel do dispositivo

Enquanto o usuário interage com o servidor de autorização, o dispositivos faz chamadas repetidas até que o processo seja concluído, o token expire ou aconteça algum outro erro, respeitando o intervalo informado.

###### 3.1 Requisição de token

O dispositivo faz uma requisição no endpoint de token, porém informando o `grant_type` como `urn:ietf:params:oauth:grant-type:device_code` e o código de dispositivo:

```http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adevice_code
&device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
&client_id=1406020730
```

###### 3.2 Resposta do token

Quando o usuário finalmente aprova e dá consentimento, o endpoint de token responde com os tokens de acordo com o informado acima [na requisição de token do fluxo de código de autorização](#requisição-de-token)

Porém existe uma extensão dos erros padrões, conforme:

|Código de erro|Descrição|Deve continuar? |
|---|---|---|
|authorization_pending|Informa que está aguardando uma interação do usuário ou ela está sendo feita no momento.|Sim|
|slow_down|Informa que ainda está aguardando uma interação do usuário, porém o dispositivo deve aumentar o intervalo das próximas chamadas em 5 segundos.|Sim|
|access_denied|O usuário recusou seu consentimento e o dispositivo não está permitido.|Não|
|expired_token|O token informado já expirou.|Não|

Quando o dispositivo recebe uma resposta de erro que não é possível continuar, então ele deve exibir alguma mensagem na tela para o usuário e deve parar de contactar o servidor para verificação.

> Note que a especificação original aborda uma hipótese de que o dispositivo não consegue receber requisições de entrada. Porém, se suportado, pode-se adotar outras abordagens, como o uso de WebSockets - dessa forma o dispositivo recebe o evento de autorização ou erro sem ter que fazer chamadas repetidas.

Se o dispositivo receber um erro de _timeout_, então é recomendado aumentar exponencialmente o intervalo para cada nova chamada com erro.

### ID Token

#### Validação do ID Token

|#|Item|Validação|
|-|---|---|
|1|_criptografado_|Se o cliente especificou uma criptografia no seu registro.<br>Se foi solicitado um token criptografado e veio um sem, deve ser rejeitado.|
|2|`iss`|O claim de emissor deve bater exatamente com o identificador do IdP, ou seja, a URL que foi chamada para autenticar, ou do serviço de descoberta.|
|3|`aud`|A claim de audiência deve conter o `client_id`, e na sua ausência deve ser rejeitado.<br>Se a lista de audiência conter outros destinos que o cliente não confia, também deve rejeitar.|
|4|`azp`|Deve existir se o token tem várias audiências/públicos.<br>Se existir deve verificar que seu `client_id` é um valor de Claim.|
|5|_TLS_|No fluxo de **Authorization Code** pode validar o TLS do servidor do IdP.|
|6|`alg`|O valor deve ser `RS256` ou o valor informado no registro pelo cliente em `id_token_signed_response_alg`.|
|7|`exp`|A data atual deve ser anterior à data de expiração do token.|
|8|`iat`|A data do claim `iat` pode ser usada para rejeitar tokens emitidos há muito tempo.|
|9|`nonce`|Se o valor do nonce foi enviado na requisição de Autenticação, então o valor deve ser exatamente o mesmo no token.|
|10|`acr`|Se o `acr` foi informado na requisição, deve-se checar se o valor é apropriado.|
|11|`auth_time`|Se o `auth_time` foi informado na requisição, ou o `max_age` então esta claim deve ser validada.<br>Deve ser solicitada uma nova autenticação se muito tempo se passou desde a última autenticação do usuário.|

#### JWT, JWE, JWS e JWTs aninhados

##### JWT

[RFC7519](https://tools.ietf.org/html/rfc7519) O resultado de 3 JSONs codificados em base64: `Cabeçalho.Corpo.Assinatura`.  
_Note que após a decodificação do JWT usando o base64, todo seu conteúdo se torna público._

##### JWS

[RFC7515](https://tools.ietf.org/html/rfc7515) O JWS é um JWT com assinatura (o mesmo que acima). Dessa forma é possível verificar que um Corpo (_payload_) não foi alterado após sua geração.  
_Obrigatório no OpenID!_

##### JWE

[RFC7516](https://tools.ietf.org/html/rfc7516) Diferente dos dois acimas, o JWE é um JWT que possui seus dados criptografados. Ou seja, não é possível ver seu conteúdo facilmente, sem o uso de uma chave.  
Segue o padrão: `Cabeçalho.Chave.Vetor.Corpo.Marca`.  
_Opcional no OpenID_

##### JWT aninhado

Ou _Nested JWT_ é um JWT dentro do outro. No OpenID geramos um JWS (o token com o payload e assinado) e então colocamos esse token dentro de outro JWT, mas criptografado, gerando um JWE.

![JWT aninhado (padrão JWS em JWE)](https://miro.medium.com/max/700/1*SuPjAL5ZN4Us1mbpZ8hGAw.png)  
_JWT aninhado (padrão JWS em JWE)_

### Claims

Os JWTs do OpenID contém várias _claims_ (reivindicações/direitos).  
Alguns são padrões e estão definidos na [RFC7519](https://tools.ietf.org/html/rfc7519).

#### Atributos padrão

|Membro|Tipo|Descrição
|---|---|---|
|`aud`|texto ou lista|Deve conter pelo menos o `client_id` do destino.|
|`exp`|número|O timestamp (segundos desde 1970) de quando o token irá expirar.|
|`auth_time`|número|Timestamp do momento em que o usuário foi autenticado.<br>Presente quando o cliente solicitou `require_auth_time`.|
|`nonce`|texto|Um valor informado na requisição para validar o uso do token na sessão.|
|`amr`|lista|Lista de identificadores de métodos de autenticação usados para autenticar o usuário. [Ver lista abaixo](#referência-de-métodos-de-autenticação)|

#### Atributos de usuário

|Membro|Tipo|Descrição
|---|---|---|
|`sub`|texto|Identificador único (no emissor) do usuário.|
|`name`|texto|Nome completo do usuário.|
|`given_name`|texto|Primeiro nome do usuário.|
|`family_name`|texto|Sobrenome do usuário.|
|`middle_name`|texto|Nome do meio do usuário (se existir).|
|`nickname`|texto|Nome casual do usuário, ou como prefere ser chamado. _ex.: `Gabs`_|
|`preferred_username`|texto|Nome curto que representa o usuário, ou o "login".<br>_Ex.:_ `gcacars` _ou_ `_g@ta.123`|
|`profile`|texto|URL da página de perfil do usuário.|
|`picture`|texto|URL da imagem de perfil **do usuário** (não uma qualquer).|
|`website`|texto|URL do Blog, site ou página na organização do usuário.|
|`email`|texto|Endereço de e-mail preferido do usuário.|
|`email_verified`|booleano|`true` se foi adotada maneiras de garantir que este e-mail é controlado por este usuário.|
|`gender`|texto|`male`, `female` ou outro valor.|
|`birthdate`|texto|Data de nascimento no formato `YYYY-MM-DD` ou `YYYY`.<br>Se o ano for `0000`, então o valor foi omitido.|
|`zoneinfo`|texto|Fuso horário do usuário. _ex.: `America/Sao_Paulo`_ [Consultar lista](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)|
|`locale`|texto|Localização ou idioma do usuário. _ex.: `pt-BR`_ [Consultar lista](https://gist.github.com/msikma/8912e62ed866778ff8cd)|
|`phone_number`|texto|Número de telefone preferencial do usuário. _ex_.: `+5511955552222`|
|`phone_number_verified`|booleano|`true` quando foi adotado maneiras de garantir que esse telefone realmente pertence à este usuário.|
|`address`|objeto JSON|Endereço postal do usuário.|
|`updated_at`|número|_Timestamp_ da última  vez que as informações foram atualizadas.|

#### Claims em vários idiomas

Para representar uma claim em vários idiomas, usa-se o formato `[claim]#[idioma][-[PAÍS]]?`, como por exemplo:

|Claim|Valor|
|---|---|
|`profile`|`https://exemplo.com.br/gabriel`|
|`profile#en-US`|`https://example.com.us/gabriel`|
|`profile#es`|`https://ejemplo.com.ar/gabriel`|

#### A claim ACR

O _Authentication Context Class Reference_ especifica um conjunto de regras de negócio que uma requisição de autenticação deve satisfazer.

Um cliente solicita no início do processo quais ACRs devem ser satisfeitos através do parâmetro `acr_values`. Ao final do processo o token ID gerado irá conter uma lista das regras que foram satisfeitas na claim `acr` (e não **como** foram).

Quando esse valor é `0`, significa que tem um nível baixo de segurança e não atende os requisitos da [ISO 29115](https://www.iso.org/standard/45138.html), ou seja, um cliente não deve permitir nenhuma movimentação monetária, por exemplo. Esse valor pode/deve estar presente no token quando por exemplo a autenticação foi feita através de um token de longo prazo de expiração (como quando há um )

#### A claim AMR

O _Authentication Methods References_ informa quais métodos de autenticação foram usados para garantir um certo nível de garantia ([RFC6711](https://tools.ietf.org/html/rfc6711)) de que o usuário informado no token é realmente o usuário da sessão, como quando após digitar a senha, o usuário ainda precisa usar sua digital no celular para confirmar o acesso. Ou quando um token OTP é necessário.

A claim `amr` presente no token irá informar uma lista de métodos utilizados na autenticação. _(veja a lista abaixo)_

#### Relação de ACR e AMR

Note que geralmente os métodos de autenticação utilizados em um fluxo de autenticação, são aqueles necessários para satisfazer o parâmetro `acr_values` informado na requisição. Dessa forma há uma ligação entre o `acr` e o `amr` recebido no token.

Note que o `amr` informa os métodos técnicos utilizados para satisfazer o processo e por isso é recomendado que não seja utilizado pelos clientes (ou _relying parties_) para permitir acesso à recursos, visto que os métodos podem variar com o tempo com a necessidade de métodos com maior segurança.

Ao contrário, sendo que possível, a aplicação (cliente) deve verificar se os valores de `acr` tem o resultado esperado para liberar acesso aos recursos, como fazer pagamentos por exemplo.

#### Referência de Métodos de Autenticação

A claim `amr` segundo a [RFC8176](https://tools.ietf.org/html/rfc8176) pode ter uma lista com os seguintes valores:

|Valor|Descrição
|---|---|
|`pwd`|Autenticação baseada em senha.|
|`pin`|Número de identificação pessoal.|
|`sms`|Confirmação de acesso via envio de um texto por SMS.|
|`tel`|Confirmação de acesso via uma ligação para um número.|
|`mfa`|Autenticação de múltiplo fator.<br>Quando estiver presente, deve incluir os métodos usados.|
|`otp`|_One-time password_. Senhas de uso único, como as geradas em Autenticadores no celular.|
|`mca`|Autenticação de múltiplo canal. Envolve mais de um canal de comunicação distinto.|
|`face`|Autenticação biométrica usando reconhecimento facial.|
|`fpt`|Autenticação biométrica usando uma digital (dedo).|
|`iris`|Autenticação biométrica usando um escaneamento de íris.|
|`retina`|Autenticação biométrica usando um escaneamento de retina.|
|`vbm`|Autenticação biométrica usando uma impressão de voz.|
|`wia`|Autenticação integrada do Windows.|
|`geo`|Uso de informação geográfica para autenticação.|
|`swk`|Prova de possessão através de um software de chave.|
|`hwk`|Prova de possessão através de um dispositivo de chave.|
|`kba`|Autenticação por base de conhecimento.|
|`sc`|Uso de cartões inteligentes.|

### UserInfo Endpoint

#### Requisição

Deve ser acessado uma rota enviando o token de acesso no cabeçalho.

```http
GET /userinfo HTTP/1.1
  Host: server.example.com
  Authorization: Bearer SlAV32hkKG
```

#### Resposta

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "sub": "248289761001",
    "name": "Fulano de Tal",
    "given_name": "Fulano",
    "family_name": "Tal",
    "preferred_username": "f.tal",
    "email": "fulanotal@exemplo.com.br",
    "picture": "http://exemplo.com.br/f.tal/eu.jpg"
}
```

#### Erro

Retorna um cabeçalho de resposta para identificar o tipo de autenticação que deve ser usado para conseguir acesso ao recurso. [WWW-Authenticate](https://developer.mozilla.org/pt-BR/docs/Web/HTTP/Headers/WWW-Authenticate)

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="openid",
  error="invalid_token",
  error_description="O token de acesso expirou!"
```

## OpenID Client Registration

Define como as aplicações cliente (_relying parties_) conseguem se registrar (castro inicial), usando a especificação [OpenID Connect Dynamic Registration](https://openid.net/specs/openid-connect-registration-1_0.html).

### Registro

|Metadado|Descrição|
|---|---|
|`redirect_uris`|Lista de URIs usados pelo cliente.<br>Deve ser idêntico ao informado na requisição de autenticação em `redirect_uri`.<br>Deve estar presente também em `sector_identifier_uri`|
|`response_types`|Uma lista que restringe ele mesmo dos tipos de resposta (`response_type`) que pode obter.<br>De: `code`, `token`, `id_token`|
|`grant_types`|Lista de auto-restrição dos tipos de `grant_type` que pode usar.<br>De: `authorization_code`, `implicit`, `refresh_token`.|
|`application_type`|O tipo de aplicação, por padrão é `web`, mas pode ser também `native`.|
|`contacts`|Lista de e-mails dos responsáveis.|
|`client_name`|Nome da aplicação que é apresentado para o usuário.|
|`logo_uri`|URL da imagem do logotipo apresentado para o usuário.|
|`client_uri`|URL da página inicial da aplicação.|
|`policy_uri`|URL da política de privacidade, informando como os dados do usuário são usados. (LGPD)|
|`tos_uri`|URL da página de termos de serviço.|
|`subject_type`|Uma lista de tipos de sujeito que a aplicação pode usar.<br>De: `public` ou `pairwise`.<br>No _nexso_ aplicações externas devem usar `pairwise` obrigatoriamente. (LGPD)|
|`default_max_age`|Validade máxima padrão da autenticação em segundos.<br>Nesse período o usuário deve se manter ativo.|
|`require_auth_time`|Booleano que indica se a claim `auth_time` deve estar presente. Por padrão é `false`.|
|`default_acr_values`|Lista que especifica os valores padrões de `acr` em ordem de preferência.|
|`initiate_login_uri`|URI em `https` que um terceiro pode usar para iniciar o login. Deve aceitar tanto os métodos `GET` e `POST` como os parâmetros `login_hint`, `iss` e `target_link_uri`.|
|`request_uris`|Lista de URIs que serão pré-registradas para essa aplicação no IdP, habilitando o cache dos dados.<br>A URI pode conter um fragmento que é o hash SHA-256 do conteúdo do arquivo usado para versionamento.|
|`sector_identifier_uri`|URL usando `https` para calcular um identificador pseudônimo para as contas pelo IdP, referenciando um arquivo JSON com a lista de URLs.<br>Relativo ao `pairwise` e deve conter as URIs que estão em `redirect_uris`.<br>Provê uma forma de alterar o `redirect_uri` sem ter que registrar novamente todos seus usuários.|
|`request_object_signing_alg`|Algoritmo usado para verificar a assinatura quando a requisição de autenticação ocorre com os dados em um JWT, tanto por valor `request` ou por referência `request_uri`.|
|`request_object_encryption_alg`|Algoritmo criptográfico para usar no JWE para criptografar o token de requisição de autenticação.|
|`token_endpoint_auth_method`|Forma de autenticação da aplicação (cliente) na requisição de token.<br>Pode ser: `client_secret_basic` (padrão), `client_secret_post`, `client_secret_jwt`, `private_key_jwt`.<br>No FAPI ainda existe: `tls_client_auth` e `self_signed_tls_client_auth`.|
|`token_endpoint_auth_signing_alg`|Algoritmo usado para assinar o JWT usado para autenticar o cliente quando o método é `private_key_jwt` ou `client_secret_jwt`.|
|`id_token_signed_response_alg`|Algoritmo usado para assinar o JWS.<br>Por padrão: `RS256`.|
|`id_token_encrypted_response_alg`|Algoritmo de criptografia usado no JWE.<br>Por padrão: `A128CBC-HS256`.|
|`userinfo_signed_response_alg`|JWS algoritmo para assinar respostas da _UserInfo_. (opcional)|
|`userinfo_encrypted_response_alg`|Algoritmo para usar no JWE para criptografar a resposta da _UserInfo_. (opcional)|
|`userinfo_encrypted_response_enc`|Algoritmo criptográfico para usar no JWE para criptografar a resposta da _UserInfo_. (opcional)|
|`jwks_uri`|URL do JSON Web Key Set da aplicação.|

Os metadados `client_name`, `tos_uri`, `policy_uri`, `logo_uri`, `client_uri` podem ter opções em outros idiomas usando o # e o código do idioma [conforme descrito acima](#claims-em-vários-idiomas).

#### Requisição de Registro

A requisição contém os metadados indicados acima.

```http
POST /connect/register HTTP/1.1
Content-Type: application/json
Accept: application/json
Host: server.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJ ...

{
 "application_type": "web",
 "redirect_uris": ["https://client.example.org/callback",
    "https://client.example.org/callback2"],
 "client_name": "My Example",
 "client_name#ja-Jpan-JP": "クライアント名",
 "logo_uri": "https://client.example.org/logo.png",
 "subject_type": "pairwise",
 "token_endpoint_auth_method": "client_secret_basic",
 "request_uris":
   ["https://client.example.org/rf.txt#qpXaRLh_n93TTR9F252ValdatUQvQiJi5BDub2BeznA"]
}
```

##### URI identificadora de setor

No campo `sector_identifier_uri` deve ser informado o caminho para um arquivo JSON.  
A requisição seria algo como:

```http
GET /file_of_redirect_uris.json HTTP/1.1
Accept: application/json
Host: other.example.net
```

E deve obter como resposta:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

[ "https://client.example.org/callback",
  "https://client.example.org/callback2",
  "https://client.other_company.example.net/callback" ]
```

#### Resposta de Registro

A resposta retorna os metadados informados na requisição e mais os valores padrões, além do ID gerado para o cliente e o segredo (se aplicável) conforme a tabela:

|Metadado|Descrição|
|---|---|
|`client_id`|O identificador único do cliente (aplicação).|
|`client_secret`|O segredo gerado para a aplicação conseguir se autenticar na requisição de Token.|
|`registration_access_token`|Um token de acesso que pode ser usada pela aplicação para usar outros métodos.|
|`registration_client_uri`|URI do local onde o cliente pode usar o token de acesso acima para executar outras operações.|
|`client_id_issued_at`|Timestamp de quando o registro foi criado.|
|`client_secret_expires_at`|Timestamp de quando o segredo irá expirar, ou `0` caso não tenha data de validade.|

```http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "client_id": "s6BhdRkqt3",
  "client_secret": "ZJYCqe3GGRvdrudKyZS0XhGv_Z45DuKhCUk0gBR1vZk",
  "client_secret_expires_at": 1577858400,
  "registration_access_token": "this.is.an.access.token.value.ffx83",
  "registration_client_uri": "https://server.example.com/connect/register?ient_id=s6BhdRkqt3",
  "token_endpoint_auth_method": "client_secret_basic",
  "application_type": "web",
  "redirect_uris": ["https://client.example.org/callback",
    "https://client.example.org/callback2"],
  "client_name": "My Example",
  "client_name#ja-Jpan-JP": "クライアント名",
  "logo_uri": "https://client.example.org/logo.png",
  "subject_type": "pairwise",
  "request_uris": ["https://client.example.org/rf.txt#qpXaRLh_n93TTR9F252ValdatUQvQiJi5BDub2BeznA"]
}
```

### Segurança do Registro

- Deve usar TLS, visto que trafega as credenciais em texto puro.
- Personificação do logotipo: um trapaceiro pode tentar imitar a tela de permissão usando o logotipo oficial. Para garantir isso podemos adotar (no contexto do _nexso_):
  - O servidor de autenticação deve exibir um alerta quando o logotipo não vem do domínio do cliente.
  - A imagem do logotipo pode ter restrição de acesso, permitindo apenas o IdP acessar a imagem.
- Restringir o acesso ao serviço apenas à rede interna, ou aplicações autorizadas.

## Gerenciamento da sessão

No OpenID a sessão se inicia quando a aplicação valida o ID Token recebido através do parâmetro `session_state`, calculado com base no ID do Cliente, na URL de origem e User Agent do navegador. Por padrão a funcionalidade de encerrar uma sessão (logout) não está ativada.

No `oidc-provider` ela pode ser ativada com o código abaixo, habilitando ainda as funcionalidades `check_session_iframe` (detecção de sessão) e `end_session_endpoint` (processo para encerrar uma sessão).

```js
const oidc = new Provider('http://localhost:3000', {
    features: {
        sessionManagement: true
    }
});
```

### iframe do próprio Cliente (RP)

Na aplicação podemos ter um iframe invisível incluso, que de tempos em tempos verifica se o usuário ainda tem sessão ativa e se ainda é o mesmo, através da troca de mensagens com o iframe enviando o código de estado recebido acima. Por segurança deve garantir que a troca de mensagens ocorre com a origem certa.

Um exemplo de código pode ser:

```js
var stat = "unchanged";
var mes = client_id + " " + session_state;
var targetOrigin = "https://server.example.com"; // Validates origin
var opFrameId = "op";
var timerID;

function check_session()   {
  var win = window.parent.frames[opFrameId].contentWindow;
  win.postMessage(mes, targetOrigin);
}

function setTimer() {
  check_session();
  timerID = setInterval(check_session, 5 * 1000);
}

window.addEventListener("message", receiveMessage, false);

function receiveMessage(e) {
  if (e.origin !== targetOrigin) {
    return;
  }
  stat = e.data;
  if (stat === "changed") {
    clearInterval(timerID);
    // e fazer as ações descritas abaixo...
  }
}

setTimer();
```

Quando detectar que uma sessão foi alterada, deve tentar uma requisição com `prompt=none` dentro do iframe para obter um novo ID Token, enviando o ID Token anterior em `id_token_hint`.

Se receber um novo basta atualizar na sessão, porém deve verificar se pertence ao mesmo usuário. Caso contrário, ou em caso de erro, deve fazer logout.

### iframe do IdP

A aplicação também deve carregar um iframe do servidor de autenticação `check_session_iframe`, e o servidor deve garantir que quem está chamando é uma origem esperada.

Quando receber uma mensagem de verificação, o iframe deve pegar o User Agent (que deve estar em um cookie ou no Local Storage) e calcular novamente o hash do `session_state`, se URL de origem ou ID do cliente não puderem ser detectados, então um `error` é enviado para a aplicação como resposta, ou se o cálculo difere `changed` é enviado demonstrando que não é a mesma sessão, e `unchanged` é enviado quando tudo está certo.

Um exemplo de código pode ser:

```js
window.addEventListener("message", receiveMessage, false);

function receiveMessage(e){ // e.data contém client_id e session_state

  var client_id = e.data.split(' ')[0];
  var session_state = e.data.split(' ')[1];
  var salt = session_state.split('.')[1];

  // se a mensagem for sintaticamente inválida retorna
  //     postMessage('error', e.origin)

  // se a mensagem vem de uma origem não esperada retorna
  //     postMessage('error', e.origin)

  // get_op_user_agent_state() é uma função definida no OP
  // que retorna o estado do login de User Agent no OP.
  // Como isso é feito, ele é quem decide.
  var opuas = get_op_user_agent_state();

  // Aqui, a session_state é calculada desse jeito particular,
  // mas é inteiramente responsabilidade do OP decidir como fazer
  // de acordo com os requerimentos definidos na especificação.
  var ss = CryptoJS.SHA256(client_id + ' ' + e.origin + ' ' + opuas + ' ' + salt) + "." + salt;

  var stat = '';
  if (session_state === ss) {
    stat = 'unchanged';
  } else {
    stat = 'changed';
  }

  e.source.postMessage(stat, e.origin);
};
```

> O estado da sessão é alterado quando ocorre algum evento significativo: entrou, saiu, adicionou uma nova sessão...

O parâmetro `check_session_iframe` deve ser fornecido pelo OpenID Provider Discovery, retornando a URL que o iframe irá carregar, aceitando requisições cross origens e usando o HTML postMessage API. Deve usar `https`.

### Logout

Descreve o processo de encerramento de sessão pela aplicação especificado em OIDC Front-Channel Logout 1.0 e OIDC Back-Channel Logout 1.0.

#### Front-channel

Esse processo ocorre no navegador do usuário, incorporando uma URL da aplicação (cliente/_Relying Party_) no IdP que irá limpar a sessão atual na aplicação.

Essa URI absoluta usando `https` é indicada no registro do cliente (`frontchannel_logout_uri`) e deve estar presente nos valores de `redirect_uris`. O IdP carrega um iframe com essa URL no processo de logout, e a página deve limpar a sessão associada, como cookies e localStorage.
O IdP enviará (configurar `frontchannel_logout_session_required` para verdadeiro) como _query parameters_ o `iss` (issuer) e `sid` (identificador da sessão) que devem ser comparados com os dados no token, e deve prevenir o cache:

```http
Cache-Control: no-cache, no-store
Pragma: no-cache
```

Exemplo:

```uri
https://rp.example.org/frontchannel_logout?iss=https://server.example.com&sid=08a5019c-17e1-4977-8f42-65a12843ea02
```

#### Back-channel

Diferentemente do front-channel esse processo tem uma comunicação direta entre o cliente (aplicação) e o servidor de autenticação ("IdP").

##### Logout no servidor de autenticação

No OpenID Connect Discovery, é preciso configurar os valores dos metadados abaixo no servidor de autenticação.

|Metadado|Descrição|
|---|---|
|backchannel_logout_supported|Valor booleano que indica o suporte da funcionalidade.|
|backchannel_logout_session_supported|Valor booleano que indica se o servidor deve enviar um `sid` (ID da sessão) no token de Logout para identificar a sessão.|

##### Logout no cliente (aplicação)

No registro do cliente deve ser informado os metadados:
|Metadado|Descrição|
|---|---|
|`backchannel_logout_uri`|URL usando `https` da aplicação que irá limpar a sessão nela mesmo quando receber um token de Logout.|
|`backchannel_logout_session_required`|Booleando que indica se deve ser enviado o `sid` no token de Logout.|

###### Logout Token

O token de logout, que deve ser assinado e pode ser criptografado e possui as claims:

|Claim|Descrição|
|---|---|
|`iss`|URL do servidor de autenticação.|
|`sub`|A identificação do sujeito/usuário.|
|`aud`|O ID de cliente da aplicação.|
|`iat`|Timestamp da data de geração do token.|
|`jti`|Identificador único deste token.|
|`events`|Um objeto JSON que contém o atributo vazio `http://schemas.openid.net/event/backchannel-logout` indicando que é um token de logout.|
|`sid`|Identificador da sessão atual.|

Exemplo:

```json
{
    "iss": "https://server.example.com",
    "sub": "248289761001",
    "aud": "s6BhdRkqt3",
    "iat": 1471566154,
    "jti": "bWJq",
    "sid": "08a5019c-17e1-4977-8f42-65a12843ea02",
    "events": {
        "http://schemas.openid.net/event/backchannel-logout": {}
    }
}
```

É recomendado que o token de logout expire no máximo em 2 minutos.

###### Validação do token de logout

Deve seguir os mesmos passos da [validação do ID Token](#validação-do-id-token) e ainda:

|Claim ou item|Validação|
|---|---|
|`sid`|Deve existir o identificador de sessão.|
|`events`|Deve ter um atributo `http://schemas.openid.net/event/backchannel-logout`.|
|`nonce`|**Não** deve existir.|
|`jti`|Verificar se outro token com o mesmo ID único já não foi usado.|
|`iss`|Comparar o emissor do token de Logout com o token de ID.|

Em caso de erro, rejeitar e retornar uma resposta HTTP 400 - Bad Request.

###### Requisição de logout

Uma requisição HTTP POST é enviada para o endereço cadastrado no registro do aplicativo com o body em `application/x-www-form-urlencoded`:

```http
POST /backchannel_logout HTTP/1.1
Host: rp.example.org
Content-Type: application/x-www-form-urlencoded

logout_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```

## Segurança

Algumas dicas e procedimentos para prevenir falhas de segurança e ataques maliciosos. [RFC6819](https://tools.ietf.org/html/rfc6819)

- Quando possível usar tokens criptografados.
- Ao criar um token novo, salvar em algum lugar o `jti`, como uma instância Redis, que deve expirar assim que o token expira. Em toda requisição, validar se aquele `jti` ainda existe no Redis. Em caso de atividade suspeita, remover o `jti` do Redis, o que irá invalidar todas as requisições, mesmo com um token válido. _("Blacklist")_
- O servidor deve usar SSL/TLS no mínimo na versão 1.2. _[Gerador de configuração da Mozilla](https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=intermediate&openssl=1.1.1d&guideline=5.6)_
- Detectar vazamentos de senhas e credenciais dos usuários em ferramentas como o [have i been pwned](https://haveibeenpwned.com/API/v3), inserindo todos os tokens na Blacklist, trocar a senha por outra aleatória, e enviar um e-mail de recuperação de senha.
- Também verificar vazamentos de contas do próprio serviço/produto.
- Rotacionar o código secreto do cliente de tempos em tempos (3 meses?).

## Fontes

Outras fontes de leitura (bibliografia):

- [Auth0 Docs](https://auth0.com/docs/get-started)
- [OpenID Blog - Financial-grade API (FAPI) Explained by an Implementer](https://fapi.openid.net/2020/02/26/guest-blog-financial-grade-api-fapi-explained-by-an-implementer/)
- [Understanding ID Token](https://darutk.medium.com/understanding-id-token-5f83f50fa02e)
- [Getting Started with oidc-provider](https://www.scottbrady91.com/OpenID-Connect/Getting-Started-with-oidc-provider)
- [ACR and AMR Values](https://digital.nhs.uk/services/nhs-identity/guidance-for-developers/detailed-guidance/acr-values)
