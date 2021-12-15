
# testes-de-software
Trabalho Pratico sobre Testes de Software

**Disciplina:** DCC/UFMG - Prática em Desenvolvimento de Software<br>
**Prof.:** Marco Tulio Valente<br>
**Aluno:** Breno de Castro Pimenta<br>
**Matrícula:** 2017114809

## Introdução
Este trabalho documenta três testes distintos (ver nota de alteração) da aplicação [Django Mail Admin](https://github.com/Bearle/django_mail_admin). Está aplicação é um gerenciador de emails construído em Django que integra as três principais bibliotecas orientadas a email do django:
* [django-mailbox](https://django-mailbox.readthedocs.io/en/latest/)
* [django-post-office](https://github.com/ui/django-post_office)
* [django-db-email-backend](https://github.com/stefanfoulis/django-database-email-backend)

## Ferramenta de teste
O motivo da escolha da aplicação Django Mail Admin é o fato dela ser extensamente testada. A principal ferramenta utilizada para realizar os testes é a subclasse [TestCase](https://docs.djangoproject.com/en/4.0/topics/testing/tools/#django.test.TestCase) da biblioteca [unittest](https://docs.python.org/3/library/unittest.html#module-unittest) de Python.
Para utilizar essa subclasse o desenvolvedor deve criar uma nova classe herdeira dessa subclasse, dessa forma é disponibilizado para ele diversos métodos de teste.

## Nota de alteração
Foi proposto a documentação de 3 testes não triviais, no entanto, este trabalho realizará a documentação de uma quantidade maior de testes, sendo que se viu mais proveitoso analisar arquivos como um todo por conterem toda a extensão de testes referentes a uma feature.

---
# Testes
## Testes do Comportamento do Backend
Estes testes são relativos ao que é chamado de backend dos emails, o qual é responsável por gerenciar o contexto dos emails (host, username, password...) para o envio. Os testes se encontram no arquivo [tests/test_backends.py](https://github.com/Bearle/django_mail_admin/blob/master/tests/test_backends.py).

**1.** A primeira parte consiste em criar uma classe para mockar os erros com o envio dos emails.
```python

class ErrorRaisingBackend(BaseEmailBackend):
    def send_messages(self, email_messages):
        raise Exception('Fake Error')
```

**2.** A segunda parte do teste consiste em inicializar a classe principal e realizar um teste simples pelo método _test_get_invalid_backend()_ ao tentar enviar um email passando como parâmetro um backend que não existe.
```python
class BackendTest(TestCase):
    # @override_settings(EMAIL_BACKEND='django.core.mail.backends.locmem.EmailBackend')
    def test_get_invalid_backend(self):
        with self.assertRaises(ValueError):
            send('from@example.com', backend='some_non_existing')

```

**3.** A terceira parte consiste de dois métodos distintos que realizam testes na função _get_backend()_ que é responsável por construir e retornar o backend do ambiente. Ambas os métodos utilizam-se do decorator [\@override_settings](https://docs.djangoproject.com/en/2.2/_modules/django/test/utils/) que permite alterar as variáveis de ambiente antes da sua execução.


```python
   @override_settings(DJANGO_MAIL_ADMIN={'EMAIL_BACKEND': 'test'})
    def test_old_settings(self):
        backend = get_backend()
        self.assertEqual(backend, 'test')

    @override_settings(DJANGO_MAIL_ADMIN={}, EMAIL_BACKEND='django_mail_admin.backends.CustomEmailBackend')
    def test_backend_fallback(self):
        backend = get_backend()
        self.assertEqual(backend, 'django_mail_admin.backends.CustomEmailBackend')
```

**4.** A quarta parte objetiva testar as classes _CustomEmailBackend()_ e _Outbox()_, sendo elas as duas principais classes construtoras de backend do sistema. O teste consiste na habilidade da classe custom capturar os dados inicializados junto a classe outbox.
O primeiro teste se dá com a inicilização de uma instância da classe _Outbox()_, em seguida se inicializa uma instância da classe _CustomEmailBackend()_, após as inicializações o primeiro teste verifica se a instância custom foi capaz de capturar os dados do outbox.
O segundo teste verifica se após deletar a instância outbox, ao tentar inicializar outra instância custom um erro ocorrerá.

```python
    def test_custom_email_backend(self):
        outbox = Outbox.objects.create(name='test', email_host='example.com',
                                       email_host_user='to@example.com', email_host_password='123456', active=True)
        backend = CustomEmailBackend()
        self.assertEqual(backend.host, outbox.email_host)
        self.assertEqual(backend.password, outbox.email_host_password)
        outbox.delete()
        with self.assertRaises(ValueError):
            backend2 = CustomEmailBackend()
```

**5.** A quinta e última parte busca testar a possibilidade de enviar um email sem declarar o backend explicitamente, ou seja, a função _send_email()_ buscará o backend setado pelo ambiente.
Novamente há o uso do decorator para desta vez estabelecer o OutboxEmailBackend como backend padrão do ambiente antes da execução do teste.
Em seguida realiza uma verificação dos emails em situação de envio, depois realiza o envio do email sem declarar de forma explicita o backend. Após o envio é verificado se o email foi enviado, tanto pelo retorno da função _send_email()_, quanto pela mudança no estado dos emails em situação de envio.

```python
    @override_settings(DJANGO_MAIL_ADMIN={}, EMAIL_BACKEND='django_mail_admin.backends.OutboxEmailBackend')
    def test_outbox_email_backend(self):
        count_before = len(OutgoingEmail.objects.all())
        sent_count = send_mail(
            subject='Test subject', message='message.',
            from_email='from@example.com', recipient_list=['to@example.com'],
            fail_silently=False)
        self.assertEqual(sent_count, 1)
        count_after = len(OutgoingEmail.objects.all())
        self.assertEqual(count_after, count_before + 1)
```



---

## Teste de Integração com padrão SMTP
O próximo teste é relativo à integração do sistema com o protocolo SMTP, o teste se encontra no arquivo [/tests/test_integration_smtp.py](https://github.com/Bearle/django_mail_admin/blob/master/tests/test_integration_smtp.py).
O objetivo deste teste é verificar se dado um contexto (servidor do email, senha e conta do email) a aplicação será capaz de enviar esses emails utilizando o padrão SMTP sem template ou com template de forma manual ou direta sem acarretar em erros.

**1.** Segue o método _setUp()_ no qual busca-se variáveis de ambiente e constrói-se o contexto para o envio do email e cria-se um objeto Outbox a partir desse contexto.
```python
class TestSmtp(TestCase):
    def setUp(self):
        self.test_smtp_server = os.environ.get('EMAIL_SMTP_SERVER')
        self.test_password = os.environ.get('EMAIL_PASSWORD')
        self.test_from_email = os.environ.get('EMAIL_ACCOUNT')
        self.test_account = self.test_from_email
        required_settings = [
            self.test_account,
            self.test_password,
            self.test_smtp_server,
            self.test_from_email,
        ]
        if not all(required_settings):
            self.skipTest(
                "Integration tests are not available without having "
                "the the following environment variables set: "
                "EMAIL_ACCOUNT, EMAIL_PASSWORD, EMAIL_SMTP_SERVER"
            )
        self.outbox = Outbox.objects.create(
            name='Integration Test SMTP',
            email_host=self.test_smtp_server,
            email_host_user=self.test_account,
            email_host_password=self.test_password,
            email_use_tls=True,
            active=True
        )
```
**2.** Segue o método _test_send_no_template()_, onde utilizando parte das váriavies de ambiente inicializados em _setUp()_ testa-se o protocolo ao enviar o email sem template pela função _django_mail_admin.mail.send()_.

```python
    def test_send_no_template(self):
        email = send(self.test_from_email, self.test_from_email, subject='Test', message='Testing',
                     priority=PRIORITY.now, backend='custom')


```
**3.** Segue o método _test_send_template()_, onde será inicializado um template teste e em seguida ocorrerão duas execuções de envio deste template. O primeiro criando o email manualmente utilizando _django_mail_admin.mail.create()_ e o segundo criando e enviando automaticamente com o uso da função _django_mail_admin.mail.create()_.

```python
    def test_send_template(self):
        # First try it manually
        template = EmailTemplate.objects.create(name='first', description='desc', subject='{{id}}',
                                                email_html_text='{{id}}')
        email = create(self.test_from_email, self.test_from_email, template=template, priority=PRIORITY.now,
                       backend='custom')
        v = TemplateVariable.objects.create(name='id', value=1, email=email)
        email = OutgoingEmail.objects.get(id=1)
        email.dispatch()

        # Then try it with send()
        email = send(self.test_from_email, self.test_from_email, template=template, priority=PRIORITY.now,
                     backend='custom', variable_dict={'id': 1})
```
