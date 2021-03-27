# Garantindo Rotas Seguras

Quando você trabalha com aplicativos que trabalham com dados sensíveis a forma tradicional é implementar algum controle no acesso de forma a identificar que o usuário possui as devidas permissões.

Em um app, por ser um produto geralmente de uso pessoal, não seria diferente sendo este um dos principais requisitos em seu backlog. De froma geral, você pensa "ok, coloco uma tela de login, guardo o token usando  alguma criptografia e passo ele no "Authorization" nos requests das apis seguras". Sim, esta é a forma mais tradicional mas (pelo menos do meu ponto de vista, você é livre para discordar e complementar essa visão) no flutter devido ao seu ciclo de vida é um pouco mais chato garantir que o usuário está autorizado e/ou que a própria sessão está ativa. Abaixo as premissas desta implementação:

* Somente usuários autenticados e com sessão ativa podem acessar as rotas seguras
* Redirecionar o usuário para telas específicas conforme o estado da autenticação/session do usuário
* Trabalhar com os seguintes estados da autenticação:
  * Uninitialized : Não inicializado, redireciona para a tela de splash. Isto pode ser feito apenas na primeira execução ou sempre que houver alguma novidade no app. Vai muito do RoadMap da aplicação. Neste artigo ele será executado sempre o app enrtar em memoria/primeira execução;
  * Unauthenticated: Não autenticado, redireciona para a tela de login
  * Authenticating: Em autenticação, geralmente ele efetuou o request para a API e está aguardando o retorno da mesma. É um estado intermediário;
  * Authenticated: Autenticado e possui token JWT. Aqui não é possivel garantir que o token tenha expirado (motivo deste artigo)
  * Expired: O token é inválido. Redirecionar para a tela de "Login Expirado!" e após clicar no "OK" redirecionar para a tela de "Login"

## Como fazer da forma simples

Se você poucas telas (aqui quero dizer poucas mesmo rs) você pode implementar no resultado de seu "Buid:Future<T>" a verificação do estado atual contra a lista de estado da autenticação e redirecionar para a tela correspondente. É obivio que está é uma solução simplista, valido para um app co duas telas, para uma POC etc. No caso de uma aplicação real, com várias telas o ideal é fazer um interceptor do response e, se o código for diferente de 200 jogar para uma classe especializada. Bom, isto funciona bem no Angular (canActivate), no React (na mão mesmo) e até n oandroid nativo mas, no Flutter, a menos que vc esteja usando um padrão mvc, mvvm, ou outro qualquer, a requisição da sua api segura foi feita na criação do seu build.
  
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

