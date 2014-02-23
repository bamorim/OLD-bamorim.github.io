---
layout: post
title: Me divertindo com trabalho de Banco de Dados
---

Estou fazendo a disciplina de Banco de Dados na universidade e um dos trabalinhos (bem bobos) que o professor passou foi de visualização de dados. Ele pediu para fazer coisas usando alguns softwares, um deles era o [Treebolic][treebolic], que gera uma visualização em java de uma árvore. A visualização é bem legal e como input ele recebe um XML num formato específico.

Eu poderia simplesmente ter usado o Treebolic Generator, que ajuda a criar essas árvores e ter entregue o trabalho, mas claro, eu quis brincar um pouco com isso. A idéia que eu tive foi usar a API do facebook para categorizar meus amigos (por mais que as categorias não sejam hierárquicas) de forma hierárquica. PEnsei também em algumas categorias que eu queria ter: Sexo, Status de Relacionamento e Signo do Zodíaco.

**TL;DR:** [Veja o resultado final.][resultado]

Vamos lá. Eu então comecei a escrever um script em [NodeJS][nodejs] para pegar os dados do facebook, hierarquizar e gerar o xml do Treebolic. Para obter os dados do facebook de forma mais simples, eu usei o [fbgraph][fbgraph], assim, pude montar o esqueleto da minha aplicação:

# Conectando com o facebook

``` javascript
var fbgraph = require( 'fbgraph' );

graph.setAccessToken( process.env.FB_ACCESS_TOKEN );

graph.get( '/me/friends?fields=gender,relationship_status,birthday,name,id' , printFriendsXml );

```

Com essa chamada eu apenas pegava os meus amigos (o limite da api do facebook é 5000 amigos por chamada!) e passava o processamento para uma função `printFriendsXml` que iria escrever o XML na tela.

# Entendendo o XML do Treebolic

Antes de prosseguir agora, eu precisava enteder como era o XML do Treebolic. Pra isso usei o Treebolic Generator e criei o default deles e analizei o XML, que é bastante simples:

``` xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!--created Sun Feb 23 02:13:07 BRT 2014--><!DOCTYPE treebolic SYSTEM "Treebolic.dtd">
<treebolic>
  <tree>
    <nodes>
      <node backcolor="ff0000" forecolor="000000" id="root">
        <label>root</label>
        <node backcolor="ffc800" forecolor="000000" id="id2">
          <label>one</label>
        </node>
        <node backcolor="ffc800" forecolor="000000" id="id2">
          <label>two</label>
        </node>
        <node backcolor="ffc800" forecolor="000000" id="id3">
          <label>three</label>
        </node>
        <node backcolor="ffc800" forecolor="000000" id="id4">
          <label>four</label>
        </node>
        <node backcolor="ffc800" forecolor="000000" id="id5">
          <label>five</label>
        </node>
      </node>
    </nodes>
  </tree>
</treebolic>
```

Como vocês podem ver, os nós da árvore começam a partir do `<nodes>` e precisa de um nó raiz. Para cada nó existe um filho `<label>` que é o texto que vai aparecer e caso tenha nós filhos, terá no mesmo nível do `<label>` outros `<node>`.

Sendo assim eu primeiro pensei em como representar isso num objeto javascript e é bem simples:

```
{
  name: "root",
  children: [/*Nós filhos*/]
}
```

# Processando os dados

Também era preciso saber como classificar por Sexo, Status de Relacionamento e Signo do Zodíaco. Para os dois primeiros é só pegar o atributo correspondente e agrupar pelos valores. Para o Signo era preciso uma função que baseado na data de nascimento eu obtivesse o signo correspondete.

Com isso eu imaginei uma função que gerasse um xml do treebolic dado um nó root. Com isso eu consegui montar a função `printFriendsXml`:

``` javascript
function printFriendsXml( err, res ){
  if(err) return console.log(err);
 
  process.stdout.write( generateTreebolicXml( {
    name: "Eu", 
    children: categorizeFriends( res.data, ['gender','relationship_status',getSignName])
  }));
};
```

# Categorizando os amigos

Com isso, a função `categorizeFriends` deve aceitar como argumentos uma array de amigos e uma array com classificadores. Estes classificadores podem ser o nome de um atributo ou uma função.

A função categorize friends eu fui mudando conforme necessário, inclusive dei umas refatoradas, mas no final, o que eu cheguei foi:

``` javascript
function categorizeFriends(friends, classificators){
  var thisClassificator = classificators[0],
      nextClassificators = classificators.slice(1),
      classifications;
  
  classifications = friends.map( classify( thisClassificator ) ).reduce( groupByClassifications, {});

  return Object.keys(classifications).map(function(klass){
    var children;
    if(nextClassificators.length > 0) {
      children = categorizeFriends(classifications[klass],nextClassificators);
    } else {
      children = classifications[klass];
    }
    return {name: klass, children: children};
  })

};
```

Basicamente recebe um grupo de amigos e um grupo de classifcadores. Depois com isso ele gera um objeto de classificações na forma de `{ classe: [filhos...]}` e depois para cada chave nessas classificações ele gera uma array na forma desejada) Isso porque os nós de amigos já tem o atributo `name` que vai ser usado para gerar as folhas da árvore. Então é uma função recursiva, para cada classificador ele gera os filhos, quando não há mais os filhos são as próprias pessoas.

A função classify gera uma outra função que serve de função de map para cada classificador, nela eu verifico se o classificador é uma string, então deverá retornar o atributo, ou se é uma função, que no caso deverá retornar o retorno dessa função. Isso gera vários objetos do tipo: `{klass: "Classificação", friend: { ... }` que depois alimentará a função de reduce `groupByClassifications`.

``` javascript
function classify(classificator,classifications){
  return function(friend){
    if(typeof classificator === "string") {
      return {klass: (friend[classificator] || "Undefined"), friend: friend};
    } else {
      return {klass: classificator(friend), friend: friend};
    }
  }
}
```

A função groupByClassifications verifica se já existe aquela classificação, se sim, coloca mais um amigo nela, se não, cria uma array com apenas o amigo em questão.

**OBS: Se alguem puder me dar uma dica de como fazer isso sem efeitos colaterais, de forma mais funcional, aceito dicas. **

``` javascript
function groupByClassifications( classifications,  node ) {
  if(node.klass in classifications) {
    classifications[node.klass].push(node.friend);
  } else {
    classifications[node.klass] = [node.friend];
  }
  return classifications;
}
```

A função de obter o signo é bem simples e declarativa:

``` javascript
function getSignName ( person ) {
  return getSign( person ).name;
}

function getSign( person ) {
  var signs = [
    {name: "Capricorn", end: "01/20"},
    {name: "Aquarius", end: "02/19"},
    {name: "Pisces", end: "03/20"},
    {name: "Aries", end: "04/20"},
    {name: "Taurus", end: "05/20"},
    {name: "Gemini", end: "06/20"},
    {name: "Cancer", end: "07/20"},
    {name: "Leo", end: "08/20"},
    {name: "Virgo", end: "09/20"},
    {name: "Libra", end: "11/20"},
    {name: "Scorpio", end: "11/20"},
    {name: "Sagittarius", end: "12/20"},
    {name: "Capricorn", end: "12/31"}
  ];

  return signs.filter( function( sign ){
    return sign.end >= person.birthday
  } )[0] || {name: "Undefined"};
};
```

# Gerando o XML

Por fim, para gerar o xml as funções são bem básicas, apenas wrappers: 

``` javascript
function generateTreebolicXml( root ) {
  return '<?xml version="1.0" encoding="UTF-8" standalone="no"?>'+
    '<!DOCTYPE treebolic SYSTEM "Treebolic.dtd">'+
    '<treebolic>'+
    '<tree>'+
    '<nodes>'+
    generateTreebolicXmlNode( {} )( root ) +
    '</nodes>'+
    '</tree>'+
    '</treebolic>';
};

function generateTreebolicXmlNode( existingIds ) {
  return function(node){
    var children = node.children || [],
        id;
    
    if(node.id) {
      id = "user-"+node.id
    } else {
      id = generateUniqueId(existingIds,node.name);
    }

    return '<node backcolor="ffffff" forecolor="000000" id="'+id+'">'+
      '<label>'+node.name+'</label>'+
      children.map( generateTreebolicXmlNode(existingIds) ).join( "" )+
      '</node>';
  };
};

function generateUniqueId( existingIds, node_name ){
  var key = node_name.toLowerCase().replace(/\s+/,"-").replace(/[^a-z0-9-]+/g,"");
  if(existingIds[key]){
    existingIds[key]++;
  } else {
    existingIds[key] = 1;
  }
  return key+"-"+existingIds[key];
}
```
Basicamente, eu primeiro envolvo com o xml base. Como todos os ids tem que ser únicos, eu fiz a função generateUniqueId que pega a quantidade de ids com a mesma chave já instanciados e coloco o número na frente -1, -2 e etc... Com isso, a função `generateTreebolicXmlNode` é "curried" para facilitar passar os ids existentes para cada iteração.

Enfim, agora vou voltar a trablhar porque já deu trabalho de mais.

# Conclusão

No final o resultado foi bem legal, por mais que eu não tivesse perdido muito tempo customizando a visualização (cores diferentes, imagens e etc).
Caso queira ver como ficou a minha árvore, [clique aqui][resultado] para ver a applet com o treebolic!

[treebolic]: http://treebolic.sourceforge.net/en/
[nodejs]: http://nodejs.org/
[fbgraph]: https://github.com/criso/fbgraph
[resultado]: /treebolic.html
