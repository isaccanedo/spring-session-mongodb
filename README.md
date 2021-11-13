# 1. Visão Geral
Neste tutorial rápido, exploraremos como usar a Spring Session com MongoDB, com e sem Spring Boot.

O Spring Session também pode ser feito com outros armazenamentos, como Redis e JDBC.

# 2. Configuração do Spring Boot
Primeiro, vamos dar uma olhada nas dependências e na configuração necessária para o Spring Boot. Para começar, vamos adicionar as versões mais recentes de spring-session-data-mongodb e spring-boot-starter-data-mongodb ao nosso projeto:

```
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-mongodb</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
```

Depois disso, para habilitar a configuração automática do Spring Boot, precisaremos adicionar o tipo de armazenamento Spring Session como mongodb em application.properties:

```
spring.session.store-type=mongodb
```

# 3. Configuração Spring sem Spring Boot
Agora, vamos dar uma olhada nas dependências e na configuração necessária para armazenar a sessão Spring no MongoDB sem Spring Boot.

Semelhante à configuração do Spring Boot, precisaremos da dependência spring-session-data-mongodb. No entanto, aqui usaremos a dependência spring-data-mongodb para acessar nosso banco de dados MongoDB:

```
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-mongodb</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-mongodb</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
```

Finalmente, vamos ver como configurar o aplicativo

```
@EnableMongoHttpSession
public class HttpSessionConfig {

    @Bean
    public JdkMongoSessionConverter jdkMongoSessionConverter() {
        return new JdkMongoSessionConverter(Duration.ofMinutes(30));
    }
}
```

A anotação @EnableMongoHttpSession permite a configuração necessária para armazenar os dados da sessão no MongoDB.

Além disso, observe que o JdkMongoSessionConverter é responsável por serializar e desserializar os dados da sessão.

# 4. Exemplo de aplicativo
Vamos criar um aplicativo para testar as configurações. Estaremos usando Spring Boot, pois é mais rápido e requer menos configuração.

Começaremos criando o controlador para lidar com as solicitações:

```
@RestController
public class SpringSessionMongoDBController {

    @GetMapping("/")
    public ResponseEntity<Integer> count(HttpSession session) {

        Integer counter = (Integer) session.getAttribute("count");

        if (counter == null) {
            counter = 1;
        } else {
            counter++;
        }

        session.setAttribute("count", counter);

        return ResponseEntity.ok(counter);
    }
}
```

Como podemos ver neste exemplo, estamos incrementando o contador em cada ocorrência no terminal e armazenando seu valor em um atributo de sessão denominado contagem.

# 5. Testando o aplicativo
Vamos testar o aplicativo para ver se podemos realmente armazenar os dados da sessão no MongoDB.

Para fazer isso, acessaremos o endpoint e inspecionaremos o cookie que receberemos. Isso conterá um id de sessão.

Depois disso, consultaremos a coleção do MongoDB para buscar os dados da sessão usando o ID da sessão:

```
@Test
public void 
  givenEndpointIsCalledTwiceAndResponseIsReturned_whenMongoDBIsQueriedForCount_thenCountMustBeSame() {
    
    HttpEntity<String> response = restTemplate
      .exchange("http://localhost:" + 8080, HttpMethod.GET, null, String.class);
    HttpHeaders headers = response.getHeaders();
    String set_cookie = headers.getFirst(HttpHeaders.SET_COOKIE);

    Assert.assertEquals(response.getBody(),
      repository.findById(getSessionId(set_cookie)).getAttribute("count").toString());
}

private String getSessionId(String cookie) {
    return new String(Base64.getDecoder().decode(cookie.split(";")[0].split("=")[1]));
}
```

6. Como funciona?
Vamos dar uma olhada no que acontece na sessão de primavera nos bastidores.

O SessionRepositoryFilter é responsável pela maior parte do trabalho:

- converte a HttpSession em MongoSession;
- verifica se há um Cookie presente e, em caso afirmativo, carrega os dados da sessão da loja;
- salva os dados da sessão atualizados na loja;
- verifica a validade da sessão.

Além disso, o SessionRepositoryFilter cria um cookie com o nome SESSION que é HttpOnly e seguro. Este cookie contém o id da sessão, que é um valor codificado em Base64.

Para personalizar o nome ou as propriedades do cookie, teremos que criar um bean Spring do tipo DefaultCookieSerializer.

Por exemplo, aqui estamos desativando a propriedade httponly do cookie:


```
@Bean
public DefaultCookieSerializer customCookieSerializer(){
    DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
        
    cookieSerializer.setUseHttpOnlyCookie(false);
        
    return cookieSerializer;
}
```

# 7. Detalhes da sessão armazenados no MongoDB
Vamos consultar nossa coleção de sessão usando o seguinte comando em nosso console MongoDB:

```
db.sessions.findOne()
```

Como resultado, obteremos um documento BSON semelhante a:

```
{
    "_id" : "5d985be4-217c-472c-ae02-d6fca454662b",
    "created" : ISODate("2019-05-14T16:45:41.021Z"),
    "accessed" : ISODate("2019-05-14T17:18:59.118Z"),
    "interval" : "PT30M",
    "principal" : null,
    "expireAt" : ISODate("2019-05-14T17:48:59.118Z"),
    "attr" : BinData(0,"rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAABdAAFY291bnRzcgARamF2YS5sYW5nLkludGVnZXIS4qCk94GHOAIAAUkABXZhbHVleHIAEGphdmEubGFuZy5OdW1iZXKGrJUdC5TgiwIAAHhwAAAAC3g=")
}
```

O _id é um UUID que será codificado em Base64 pelo DefaultCookieSerializer e definido como um valor no cookie SESSION. Além disso, observe que o atributo attr contém o valor real do nosso contador.

# 8. Conclusão
Neste tutorial, exploramos o Spring Session com MongoDB - uma ferramenta poderosa para gerenciar sessões HTTP em um sistema distribuído. Com esse propósito em mente, pode ser muito útil para resolver o problema de replicação de sessões em várias instâncias do aplicativo.