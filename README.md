### Servidor HTTP com Netty

# 1. Introdução
Neste tutorial, vamos implementar um servidor simples com letras maiúsculas sobre HTTP com Netty, uma estrutura assíncrona que nos dá flexibilidade para desenvolver aplicativos de rede em Java.

# 2. Bootstrap do servidor
Antes de começar, devemos estar cientes dos conceitos básicos do Netty, como canal, manipulador, codificador e decodificador.

Aqui, vamos pular direto para a inicialização do servidor, que é basicamente o mesmo que um servidor de protocolo simples:

```
public class HttpServer {

    private int port;
    private static Logger logger = LoggerFactory.getLogger(HttpServer.class);

    // constructor

    // main method, same as simple protocol server

    public void run() throws Exception {
        ...
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup)
          .channel(NioServerSocketChannel.class)
          .handler(new LoggingHandler(LogLevel.INFO))
          .childHandler(new ChannelInitializer() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline p = ch.pipeline();
                p.addLast(new HttpRequestDecoder());
                p.addLast(new HttpResponseEncoder());
                p.addLast(new CustomHttpServerHandler());
            }
          });
        ...
    }
}
```

Então, aqui apenas o childHandler difere de acordo com o protocolo que queremos implementar, que é HTTP para nós.

Estamos adicionando três manipuladores ao pipeline do servidor:

- HttpResponseEncoder do Netty - para serialização;
- HttpRequestDecoder do Netty - para desserialização;
- Nosso próprio CustomHttpServerHandler - para definir o comportamento do nosso servidor.

Vamos examinar o último manipulador em detalhes a seguir.

# 3. CustomHttpServerHandler
O trabalho do nosso manipulador personalizado é processar os dados de entrada e enviar uma resposta.

Vamos decompô-lo para entender seu funcionamento.

### 3.1. Estrutura do Handler
CustomHttpServerHandler estende o SimpleChannelInboundHandler abstrato do Netty e implementa seus métodos de ciclo de vida:

```
public class CustomHttpServerHandler extends SimpleChannelInboundHandler {
    private HttpRequest request;
    StringBuilder responseData = new StringBuilder();

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) {
       // implementation to follow
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

Como o nome do método sugere, channelReadComplete libera o contexto do manipulador após a última mensagem no canal ter sido consumida para que esteja disponível para a próxima mensagem de entrada. O método exceptionCaught é para lidar com exceções, se houver.

Até agora, tudo o que vimos é o código clichê.

Agora vamos continuar com as coisas interessantes, a implementação de channelRead0.

### 3.2. Lendo o canal
Nosso caso de uso é simples, o servidor simplesmente transformará o corpo da solicitação e os parâmetros da consulta, se houver, em maiúsculas. Uma palavra de cautela aqui ao refletir os dados da solicitação na resposta - estamos fazendo isso apenas para fins de demonstração, para entender como podemos usar o Netty para implementar um servidor HTTP.

Aqui, consumiremos a mensagem ou solicitação e configuraremos sua resposta conforme recomendado pelo protocolo (observe que RequestUtils é algo que escreveremos em alguns instantes):

```
if (msg instanceof HttpRequest) {
    HttpRequest request = this.request = (HttpRequest) msg;

    if (HttpUtil.is100ContinueExpected(request)) {
        writeResponse(ctx);
    }
    responseData.setLength(0);            
    responseData.append(RequestUtils.formatParams(request));
}
responseData.append(RequestUtils.evaluateDecoderResult(request));

if (msg instanceof HttpContent) {
    HttpContent httpContent = (HttpContent) msg;
    responseData.append(RequestUtils.formatBody(httpContent));
    responseData.append(RequestUtils.evaluateDecoderResult(request));

    if (msg instanceof LastHttpContent) {
        LastHttpContent trailer = (LastHttpContent) msg;
        responseData.append(RequestUtils.prepareLastResponse(request, trailer));
        writeResponse(ctx, trailer, responseData);
    }
}
```

Como podemos ver, quando nosso canal recebe um HttpRequest, ele primeiro verifica se a solicitação espera um status 100 Continue. Nesse caso, respondemos imediatamente com uma resposta vazia com o status CONTINUE:

```
private void writeResponse(ChannelHandlerContext ctx) {
    FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, CONTINUE, 
      Unpooled.EMPTY_BUFFER);
    ctx.write(response);
}
```

Depois disso, o manipulador inicializa uma string a ser enviada como uma resposta e adiciona os parâmetros de consulta da solicitação a ela para ser enviada de volta como está.

Vamos agora definir o método formatParams e colocá-lo em uma classe auxiliar RequestUtils para fazer isso:

```
StringBuilder formatParams(HttpRequest request) {
    StringBuilder responseData = new StringBuilder();
    QueryStringDecoder queryStringDecoder = new QueryStringDecoder(request.uri());
    Map<String, List<String>> params = queryStringDecoder.parameters();
    if (!params.isEmpty()) {
        for (Entry<String, List<String>> p : params.entrySet()) {
            String key = p.getKey();
            List<String> vals = p.getValue();
            for (String val : vals) {
                responseData.append("Parameter: ").append(key.toUpperCase()).append(" = ")
                  .append(val.toUpperCase()).append("\r\n");
            }
        }
        responseData.append("\r\n");
    }
    return responseData;
}
```

Em seguida, ao receber um HttpContent, pegamos o corpo da solicitação e o convertemos em maiúsculas:

```
StringBuilder formatBody(HttpContent httpContent) {
    StringBuilder responseData = new StringBuilder();
    ByteBuf content = httpContent.content();
    if (content.isReadable()) {
        responseData.append(content.toString(CharsetUtil.UTF_8).toUpperCase())
          .append("\r\n");
    }
    return responseData;
}
```

Além disso, se o HttpContent recebido for um LastHttpContent, adicionamos uma mensagem de despedida e cabeçalhos finais, se houver:

```
StringBuilder prepareLastResponse(HttpRequest request, LastHttpContent trailer) {
    StringBuilder responseData = new StringBuilder();
    responseData.append("Good Bye!\r\n");

    if (!trailer.trailingHeaders().isEmpty()) {
        responseData.append("\r\n");
        for (CharSequence name : trailer.trailingHeaders().names()) {
            for (CharSequence value : trailer.trailingHeaders().getAll(name)) {
                responseData.append("P.S. Trailing Header: ");
                responseData.append(name).append(" = ").append(value).append("\r\n");
            }
        }
        responseData.append("\r\n");
    }
    return responseData;
}
```

### 3.3. Escrevendo a resposta
Agora que nossos dados a serem enviados estão prontos, podemos escrever a resposta ao ChannelHandlerContext:

```
private void writeResponse(ChannelHandlerContext ctx, LastHttpContent trailer,
  StringBuilder responseData) {
    boolean keepAlive = HttpUtil.isKeepAlive(request);
    FullHttpResponse httpResponse = new DefaultFullHttpResponse(HTTP_1_1, 
      ((HttpObject) trailer).decoderResult().isSuccess() ? OK : BAD_REQUEST,
      Unpooled.copiedBuffer(responseData.toString(), CharsetUtil.UTF_8));
    
    httpResponse.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain; charset=UTF-8");

    if (keepAlive) {
        httpResponse.headers().setInt(HttpHeaderNames.CONTENT_LENGTH, 
          httpResponse.content().readableBytes());
        httpResponse.headers().set(HttpHeaderNames.CONNECTION, 
          HttpHeaderValues.KEEP_ALIVE);
    }
    ctx.write(httpResponse);

    if (!keepAlive) {
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
    }
}
```

Neste método, criamos um FullHttpResponse com a versão HTTP / 1.1, adicionando os dados que preparamos anteriormente.

Se uma solicitação deve ser mantida ativa, ou em outras palavras, se a conexão não deve ser fechada, definimos o cabeçalho de conexão da resposta como keep-alive. Caso contrário, fechamos a conexão.

# 4. Testando o servidor
Para testar nosso servidor, vamos enviar alguns comandos cURL e ver as respostas.

Claro, precisamos iniciar o servidor executando a classe HttpServer antes disso.

### 4.1. Solicitação GET
Vamos primeiro invocar o servidor, fornecendo um cookie com a solicitação:

```
curl http://127.0.0.1:8080?param1=one
```

Como resposta, obtemos:

```
Parameter: PARAM1 = ONE

Good Bye!
```

Também podemos acessar http://127.0.0.1:8080?param1=one de qualquer navegador para ver o mesmo resultado.

### 4.2. Solicitação POST
Como nosso segundo teste, vamos enviar um POST com o conteúdo da amostra do corpo:

```
curl -d "sample content" -X POST http://127.0.0.1:8080
```

Aqui está a resposta:

```
SAMPLE CONTENT
Good Bye!
```

Desta vez, como nossa solicitação continha um corpo, o servidor o enviou de volta em maiúsculas.

# 5. Conclusão
Neste tutorial, vimos como implementar o protocolo HTTP, particularmente um servidor HTTP usando Netty.

HTTP/2 no Netty demonstra uma implementação cliente-servidor do protocolo HTTP/2.