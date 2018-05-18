---
title: "Aula dois em um de django: Custom User e Usando Signals de campos ManyToMany"
layout: post
date: 2018-05-17 21:36
headerImage: false
tag:
- Django
- Python
- Intermediário
category: blog
author: jlugao
description: Vamos descobrir como criar um modelo de usuário personalizado em django.
Além disso vamos ver como disparar eventos toda vez que um campo many-to-many for
atualizado e usar essa informação para fazer alguma coisa em nosso projeto
---

Fala pessoal,

Como eu só criei um post nesse blog até hoje e isso aqui tá super abandonado, resolvi
criar um post dobradinha: vou ensinar vocês duas coisas importantes que aprendi em django.

Para estabelecer uma motivação, vamos imaginar que queremos implementar um sistema de cursos online, nesse sistema, teremos cursos em que os alunos podem se inscrever. Queremos que quando o usuário se matricular em um curso novo, que esse vire seu curso padrão para que ele possa iniciar as aulas rapidamente.

Vamos começar falando de usuário. O Django vem com um modelo de usuário embutido
muito, mas muito massa mesmo. Mas as vezes a gente quer colocar mais informações nesse cadastro
de usuário, um jeito muito comum de resolver isso é criando um modelo com um campo one-to-one para
o modelo de usuário, dessa forma, esse novo modelo serviria de perfil para este usuário.
Não há nenhum problema nisso, na verdade é um jeito muito rápido e eficaz de inserir mais informações nos usuários.

Mas se você quiser um pouco mais de controle, você pode utilizar a classe `AbstractUser`, como eu vou mostrar abaixo. Basta seguir dois passos importantes:

1 - Criar um modelo herdando da classe `AbstractUser` (numa app chaada `learning`)

```python
from django.db import models

from django.contrib.auth.models import AbstractUser

#... modelo de cursos irá entrar aqui

class User(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    current_course = models.ForeignKey(Course, blank=True, null=True,
                                        on_delete=models.DO_NOTHING)
    def __str__(self):
        return self.first_name + ' '  + self.last_name
```

2 - Nas configurações do seu projeto colocar uma referência para o novo modelo de usuário como mostrado abaixo
```
AUTH_USER_MODEL = 'learning.User'
```

Bem simples, não? já temos nosso novo modelo de usuário.

Agora vamos criar um modelo de curso bem simples:

```python
class Course(models.Model):
    title = models.TextField(max_length=100)
    users = models.ManyToManyField('learning.User', blank=True)
    def __str__(self):
        return self.title

# ... modelo de Usr.
```

Ok, como podemos ver o modelo que representa o curso tem uma referência do tipo ManyToMany para o usuário, representando todos os usuários que podem estar matriculados em qualquer curso (e todos os cursos aos quais os usuários podem estar matriculados). O usuário por sua vez possui uma referência para o curso atual.

Agora tudo o que a gente precisa fazer é mudar o campo `current_course` toda vez que matricularmos o usuário em um curso. Quando eu me encontrei numa situação parecida eu quebrei muito a cabeça e pesquisei muito para encontrar a solução (que achei elegantissima) abaixo:
```python
from django.db.models.signals import m2m_changed
from django.dispatch import receiver

@receiver(m2m_changed,sender=Course.users.through)
def cadastrado_no_projeto(sender, instance, model, pk_set, **kwargs):
    model.objects.filter(pk__in=pk_set).update(current_course=instance)
```
O que está acontecendo aqui? Nós conseguimos criar um receptor para os sinais emitidos pelo modelo da tabela intermediária criada por baixo dos panos pelo campo ManyToMany.

Aí a gente precisa esclarecer algumas coisas, quando esse sinal é disparado `instance` representa a instancia de origem (o curso que a gente salvou no caso), `model` representa o modelo de destino da relação (nesse caso o nosso modelo de usuário personalizado), `pk_set`, as `pks` dos itens atualizados do modelo de destino.

Dessa forma, filtramos os objetos do modelo que pertençam ao grupo de pks dos objetos que foram atualizados. A partir desse conjunto filtrado, atualizamos o campo `current_course` com o curso que deu origem a todos esses sinais.

Pode ser que você demore um pouco para entender tudo isso que está acontecendo nessas poucas linhas, eu sei que eu demorei um pouco. Mas é uma forma muito prática e até elegante de resolver esse problema.

Para poder brincar um pouco mais com isso, sugiro registrar os modelos no admin.py e testar.

```python
from django.contrib import admin
from .models import User, Course
# Register your models here.
admin.site.register(User)
admin.site.register(Course)
```

Por hoje é só galera, espero que tenham ficados claros os conceitos que eu queria passar, qualquer dúvida posta aí que eu tento ajudar vocês. Ah, vamos ver se eu faço um post explicando melhor sinais em geral, mas para esse tema e modelo de usuário recomendo muito a leitura do blog Simple Is Better Than Complex. Que traz leituras excelentes sobre o tema.

Abraço,
João
