---
layout: post
title:  "O que aprendi hoje: git update-index --assume-unchanged"
date:   2016-11-10 23:22:47 -0200
categories: git update-index
---
Uma coisa que sempre me incomoda é alterar um arquivo versionado pelo repositório, como por exemplo um arquivo de configuração e sempre que executo o `git status` esse arquivo ser listado.

Hoje aprendi que o git possui um comando chamado **update-index** que tem por finalidade manipular o *index* do repositório. Muitas das operações presentes no **update-index** também são encontradas de uma maneira mais amigável no comando **add**. Referências para o **update-index** e **add** podem ser encontrados [aqui][update-index-ref] e [aqui][add-ref].

Com o **update-index** podemos resolver o problema dos configs usando as flags **\--assume-unchanged** e **\--no-assume-unchanged**. Ao utilizar a flag **\--assume-unchanged** passando um arquivo ou diretório como parâmetro o *index* ira assumir que este caminho não será mais modificado, assim não será mais listado como alteração no `git status`. Para modificar o arquivo ou diretório novamente devemos remover com o flag **\--no-assume-unchanged**.

Exemplos:

{% highlight shell %}
$ git status
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   app/controllers/application_controller.rb
        modified:   config/database.yml

no changes added to commit (use "git add" and/or "git commit -a")

$ git update-index --assume-unchanged config/database.yml
{% endhighlight %}

Se executarmos o `git status` o arquivo `config/database.yml` não será mais listado.

{% highlight shell %}
$ git status
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   app/controllers/application_controller.rb

no changes added to commit (use "git add" and/or "git commit -a")
{% endhighlight %}

Caso tenhamos feito várias alterações, podemos até mesmo usar o `git add .` e o arquivo `config/database.yml` não irá pro commit.

Importante lembrar que mesmo o arquivo sendo alterado novamente ele não será listado para entrar no commit. Para isso precisamos remover o flag:

{% highlight shell %}
$ git update-index --no-assume-unchanged config/database.yml
{% endhighlight %}

[update-index-ref]: https://git-scm.com/docs/git-update-index
[add-ref]: https://git-scm.com/docs/git-add
