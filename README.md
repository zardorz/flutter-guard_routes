# Garantindo Rotas Seguras

Quando você trabalha com aplicativos que trabalham com dados sensíveis a forma tradicional é implementar algum controle no acesso de forma a identificar que o usuário possui as devidas permissões.

Em um app, por ser um produto geralmente de uso pessoal, não seria diferente sendo este um dos principais requisitos em seu backlog. De forma geral, você pensa "ok, coloco uma tela de login, guardo o token usando protegido por alguma criptografia (ou serviço como o firebase) e passo ele no "Authorization" nos requests das apis seguras". Sim, esta é a forma mais utilizada mas (pelo menos do meu ponto de vista, você é livre para discordar e complementar essa visão) no flutter devido ao seu ciclo de vida é um pouco mais "chato" garantir que o usuário está autorizado e/ou que a própria sessão está ativa.  

Abaixo as premissas desta implementação:

* Somente usuários autenticados e com sessão ativa podem acessar as rotas seguras
* Redirecionar o usuário para telas específicas conforme o estado da autenticação/session do usuário
* Trabalhar com os seguintes estados da autenticação:
  * Uninitialized : Não inicializado, redireciona para a tela de splash. Isto pode ser feito apenas na primeira execução ou sempre que houver alguma novidade no app. Vai muito do RoadMap da aplicação. Neste artigo ele será executado sempre o app enrtar em memoria/primeira execução;
  * Unauthenticated: Não autenticado, redireciona para a tela de login
  * Authenticating: Em autenticação, geralmente ele efetuou o request para a API e está aguardando o retorno da mesma. É um estado intermediário;
  * Authenticated: Autenticado e possui token JWT. Aqui não é possivel garantir que o token tenha expirado (motivo deste artigo)
  * Expired: O token é inválido. Redirecionar para a tela de "Login Expirado!" e após clicar no "OK" redirecionar para a tela de "Login"

## Como fazer da forma simples

Se você poucas telas (aqui quero dizer poucas mesmo) você pode implementar no resultado de seu "Buid:Future<T>" a verificação do estado atual contra a lista de estado da autenticação e redirecionar para a tela correspondente. É obvio que está é uma solução simplista, válido para um app com poquissimas telas, para uma POC etc. No caso de uma aplicação real, com várias telas o ideal é fazer um interceptor do response e, se o StatusCode for diferente de "200-OK" jogar para uma classe especializada. Bom, isto funciona bem no Angular (canActivate), no React (na mão mesmo) e até no android mas, no Flutter, a menos que vc esteja usando um padrão mvc, mvvm, ou outro qualquer, a requisição da sua api segura foi feita na criação do seu build.
  
 ```dart
 class _StatementsView extends State<StatementsView> {
  Future<List<Statement>> statementsFuture;
  .....
  
  @override
  void didChangeDependencies() {
    statementsRepository = Provider.of(context);
    statementsFuture = getAllStatements();
    super.didChangeDependencies();
  }
  
  Future<List<Statement>> getAllStatements() async {
    return this.statementsRepository.getAll('10', '202011');
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Lançamentos"),
      ),
      body: FutureBuilder<List<Statement>>(
        future: statementsFuture,
        builder: (context, snapshot) {
          if (snapshot.hasData) {
....

```

Ou seja, a sua tela foi montada e você terá que fazer a gestão do retorno com a tela em exibição e pior, para cada tela do seu app. 

A solução prosposta ajuda a resolver esta e outras questões (aguarde outros artigos rs) de uma forma elegante, relativemnte simples, e com implementação em apenas um lugar. Claro, você pode usar o DIO ou outro objeto como o http_interceptor mas eles resolvem apenas uma parte do problema.

Bom, vamos a solução proposta

## Arquitetura

Abaixo um fluxo da arquitetura implementada

![image](https://user-images.githubusercontent.com/10378151/112729430-cc011500-8f0a-11eb-8e89-22cc5a56de98.png)

Quando o app é iniciado a classe main é executada por padrão, dentro do fluxo normal. Nesta classe é realizado a inicialização dos providers (repositórios), da classe Auth (responsável pelo controle da sessão e armazenagem segura do JWT) e a atribuição da lista de rotas ao "MaterialApp". 

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {

    return FutureBuilder(
      future: Auth.create(),
      builder: (BuildContext context, AsyncSnapshot snapshot) {
        if (Auth() == null) {
          return Container(
            child: Center(
              child: CircularProgressIndicator(
                strokeWidth: 2,
              ),
            ),
          );
        } else {
          return MultiProvider(
            providers: [
              ChangeNotifierProvider(
                create: (_) => CategoriesRepository(),
              ),
              ChangeNotifierProvider(
                create: (_) => FinanceAccountsRepository(),
              ),
              ChangeNotifierProvider(
                create: (_) => Auth(),
              ),
              ChangeNotifierProvider(
                create: (ctx) => StatementsRepository(),
              ),
              ChangeNotifierProvider(
                create: (ctx) => UsersRepository(),
              ),
            ],
            child: MaterialApp(
              debugShowCheckedModeBanner: false,
              title: 'Flutter Demo',
              theme: theme(), 
              routes: routes, 
            ),
          );
        }

```
Uma pergunta interessante seria "porque não fazer a gestão de rotas seguras pelo atributo "routes", de uma forma semelhante ao processo do react?"

```javascript
	render() {
	  return (
	    <div className="App">
		<Switch>
		  <Route exact path="/login" component={LoginPage} />
		  <ProtectedRoute path="/" component={MainPage} />
		  <Route path="*" component={() => '404 NOT FOUND'} />
		</Switch>
	    </div> 
	);

```
Então, até deve ser possível mas não encontrei nenhuma forma de efetuar isto. A principal razão é que antes de executar as rotas e os métodos seguros eu quero executar um EndPoint (EP) "IsAlive" onde o meu JWT é validado. E onde está o problema? É que o EP é executado de forma asyncrona e o atributo "routes" exige um retorno sincrono (se vc souber como fazer manda o link ;))

Voltando ao fluxo...por padrão é executada a rota "/home" ( ou "/"  conforme sua configuração).  Pela análise inicial do código é possível observar que a mesma executa um Future<bool> do "IsAlive" que é um EP seguro exigindo o JWT. Se o mesmo tiver expirado retorna false, caso contrário true.
 
 ```javascript
 return FutureBuilder<bool>(
        future: isAliveFuture,
        builder: (BuildContext context, snapshot) {
          if (snapshot.hasData) {
            return validSession();
          } else if (snapshot.hasError) {
            return Text("${snapshot.error}");
          }
          
```          
 e o método "validSession" efetua a orquestração necessária
 
 ```dart
   WillPopScope validSession() {
    switch (auth.userStatus) {
      case Status.Unauthenticated: 
          SchedulerBinding.instance.addPostFrameCallback((_) {
            Navigator.pushReplacement(context, MaterialPageRoute(builder: (BuildContext context) => SignInView()));
          }); 

        break; 
      case Status.Expired: 
          SchedulerBinding.instance.addPostFrameCallback((_) {
            Navigator.pushReplacement(context, MaterialPageRoute(builder: (BuildContext context) => ExpiredSessionView()));
          }); 
        break;

      default:
        break;
    }

    return _willPopScope();
  }
 ```
Antes que você comece a jogar pedras e me chamr de gerege, sim, todos os EP's sensíveis validam o tokem, a questão aqui é naço desejar que a tela faca a gestão do retorno 401, 404, 500,... e sim um processo controlado, único e de fácil extensão, que n~çao gere overhead ou chamadas duplas tanto da api quanto de telas (rotas). Tudo tem desvantagens mas o foco deste artigo é mostrar as vantagens que identifiquei usando esta arquitetura. Ela ainda vai ser evoluida, ainda está em sua primeira versão ;)

Ok, até agora está tudo certo mas, como é feito a mudança no estado da sessão pela simples chamada do "IsAlive"? Aqui tem outra pegadinha (como falei acima, vc pode usar outras formas). Foi criada uma classe "AuthenticatedHttpClient" que extende "http.BaseClient" onde todos os repositórios instanciam para a chamada dos EP's. No método "send" ela "injeta" o JWT e efetua o "request" na API. Como retorno temos um "StreamedResponse" que permite ler o "statusCode". Agora ficou fácil não? Poderiamos dizer que é uma forma "invertida" de colocar um interceptor (ok, eu sei que não é rs) visto que o objeto http padrão do Dart não fornece uma forma direta para isto.
 
Lendo o "StatusCode" você pode fazer a orquestração do que desejar em um único local.


 ```dart
class AuthenticatedHttpClient extends http.BaseClient {

  @override
  Future<http.StreamedResponse> send(http.BaseRequest request) async {
    var acc =  Auth().accessToken; 

    request.headers.putIfAbsent('Accept', () => 'application/json');
    request.headers.putIfAbsent('content-type', () => 'application/json');
    request.headers.putIfAbsent('Authorization', () => acc);

    final StreamedResponse response = await request.send();

    if (response.statusCode == 401) await new Auth().expireSession();

    return response;
  }
}
 ```

![image](https://user-images.githubusercontent.com/10378151/112729440-da4f3100-8f0a-11eb-871f-8eec8facbc56.png)


De uma forma "bem resumida" é isto. Ainda tremos várias refatorações neste princípio mas o caminho é este. E você como fez a gestão de seuas rotas?

