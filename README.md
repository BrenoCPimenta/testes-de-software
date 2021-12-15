
# testes-de-software
Trabalho Pratico sobre Testes de Software

**Disciplina:** DCC/UFMG - Prática em Desenvolvimento de Software
**Prof.:** Marco Tulio Valente
**Aluno:** Breno de Castro Pimenta
**Matrícula:** 2017114809

## Introdução
Este trabalho documenta três testes distintos da aplicação [Django Mail Admin](https://github.com/Bearle/django_mail_admin). Está aplicação é um gerenciador de emails construído em Django que integra as três principais bibliotecas orientadas a email do django:
* [django-mailbox](https://django-mailbox.readthedocs.io/en/latest/)
* [django-post-office](https://github.com/ui/django-post_office)
* [django-db-email-backend](https://github.com/stefanfoulis/django-database-email-backend)

## Ferramenta de teste
O motivo da escolha da aplicação Django Mail Admin é o fato dela ser extensamente testada. A principal ferramenta utilizada para realizar os testes é a subclasse [TestCase](https://docs.djangoproject.com/en/4.0/topics/testing/tools/#django.test.TestCase) da biblioteca [unittest](https://docs.python.org/3/library/unittest.html#module-unittest) de Python.
Para utilizar essa subclasse o desenvolvedor deve criar uma nova classe herdeira dessa subclasse, dessa forma é disponibilizado para ele diversos métodos de teste.


## Testes
### Teste de Integração
O próximo teste é relativo à integração do sistema com o protocolo SMTP, o teste se encontra no arquivo [/tests/test_integration_smtp.py](https://github.com/Bearle/django_mail_admin/blob/master/tests/test_integration_smtp.py).
O objetivo deste teste é verificar se dado um contexto construído pelo método _setUp()_: servidor do email, senha e conta do email, inicializados na classe Outbox, a aplicação será capaz de enviar esses emails sem template ou com template de forma manual ou direta sem acarretar em erros.

**1.** Segue o método _setUp()_ no qual busca-se o contexto em variáveis de ambiente e cria-se um objeto Outbox para o envio dos emails.
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
