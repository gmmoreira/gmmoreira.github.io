---
layout: post
title:  "O que aprendi hoje: git update-index --assume-unchanged"
date:   2016-11-10 23:22:47 -0200
categories: git update-index
---
Quando trabalhamos em um projeto com git provavelmente temos um arquivo .gitignore com uma lista de arquivos ou diretórios que devem ser ignorados, ou seja, não serão versionados.

Uma coisa que sempre me incomoda é alterar um arquivo versionado pelo repositório, como por exemplo um arquivo de configuração e todas as vezes que executo o `git status` esse arquivo ser listado.

Hoje aprendi que o git possui um comando chamado **update-index** que tem por finalidade manipular o index do repositório. Muitas das operações presentes no **update-index** também são encontradas de uma maneira mais amigável no comando **add**. Referências para o **update-index** e **add** podem ser encontrados [aqui][update-index-ref] e [aqui][add-ref].

Continuando sobre o update-index, podemos resolver o problema dos configs usando as flags **\--assume-unchanged** e **\--no-assume-unchanged**. Ao utilizar a flag **\--assume-unchanged** passando um arquivo ou diretório como parâmetro o *index* ira assumir que este caminho não será mais modificado, assim não será mais listado como alteração no `git status`. Para modificar o arquivo ou diretório novamente devemos remover com o flag **\--no-assume-unchanged**.

Exemplos:
{% highlight shell %}
git update-index --assume-unchanged config/database.yml
{% endhighlight %}

{% highlight shell %}
git update-index --no-assume-unchanged config/database.yml
{% endhighlight %}

[update-index-ref]: https://git-scm.com/docs/git-update-index
[add-ref]: https://git-scm.com/docs/git-add
