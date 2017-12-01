# Hora de especificar

Antes de desenvolver qualquer coisa, vamos conhecer a primeira iteração a ser entregue
no nosso sistema de gestão de cinema:

> * Listar filmes em exibição
> * Exibir detalhes de um filme específico

Independente do processo de desenvolvimento (cascata, iterativo, ágil), é muito importante
termos um próximo passo atingível em mente, para não desviarmos o foco.

Como estamos objetivando uma API de dados, e o desenvolvimento do front-end
ocorrerá de maneira separada (ou seja, com outra equipe) do back-end, o mais importante
dessa primeira reunião de especificação é definir a interface entre as equipes, justamente
os *endpoints* da API REST. Assim as equipes podem seguir de maneira paralela, mas claro
mantendo contato frequente para aprumar aquilo que não ficou bem aparado na primeira
definição.

Com isso em mente, vamos procurar por *recursos REST* candidatos. Fica claro que temos
um recurso REST chamado *filmes*. Vamos começar por ele então:

```
GET /filmes/em-exibicao
GET /filmes/{id}
```

Note que isso ainda está muito abstrato, precisamos também definir as *respostas
esperadas* para cada endpoint:

* `GET /filmes/em-exibicao`
  * 200 OK: `[ Filme ]`
  * 500 SERVER ERROR
* `GET /filmes/{id}`
  * 200 OK: `Filme`
  * 404 NOT FOUND
  * 500 SERVER ERROR

Mas o que significam as expressões `[ Filme ]` e `Filme`? Respectivamente, a primeira
diz que o endpoint retorna um `array` de objetos do tipo `Filme` enquanto a segunda
diz que o endpoint retorna uma única instância de objeto do tipo `Filme`. Note a
similaridade com a definição de objetos JavaScript, isso não é coincidência, pois o
formato mais utilizado para transferência de dados em APIs REST é o JavaScript Object
Notation (JSON).

Para finalizar temos que definir o tipo `Filme`:

```
Filme = {
    id: GUID,
    nome: String,
    sinopse: String,
    duracao: String, # HH:MM, uma outra opção seria um número equivalente ao total de minutos
    inicioExibicao: Date,
    fimExibicao: Date
}
```

Poderíamos já começar a discutir outros recursos, como *curtidas*, *gêneros* etc, mas como
nosso escopo não prevê nada disso ainda, vamos deixar para as próximas iterações.

Com isso a equipe de front-end já consegue andar com seu trabalho, enquanto a equipe de
back-end faz o mesmo do outro lado. [Clique aqui](04_versionando_no_github.md)
para ir para o próximo tópico.
