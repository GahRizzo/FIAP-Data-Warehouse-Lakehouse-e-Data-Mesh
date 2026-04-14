# 01 - Setup e configuração de ambiente

Este laboratório prepara todo o ambiente usado ao longo da disciplina.

## Objetivo

Ao final deste setup, você terá:

- uma conta ativa no AWS Academy
- um Codespaces pronto para executar os laboratórios
- as credenciais da AWS configuradas no Codespaces
- uma chave SSH configurada para os laboratórios que precisarem dela
- um bucket `base-config-<SEU RM>` criado no S3

## O que você vai usar

Você irá utilizar 2 ferramentas para desenvolver os laboratórios:

1. Conta AWS através da AWS Academy. Conta provisionada para você utilizar durante essa disciplina com 50 dólares de crédito.
2. [GitHub Codespaces](https://github.com/features/codespaces). Uma IDE cloud online para todos terem um ambiente igual e com autorização para executar os comandos dos exercícios.

> [!IMPORTANT]
> Guarde este material. Você vai reutilizar parte dele em toda aula, principalmente a atualização das credenciais no Codespaces.

---

## Parte 1 - Criando o GitHub Codespaces

### Resultado esperado desta parte

Ao final desta etapa, você terá um Codespaces criado a partir do repositório da disciplina e pronto para uso.

1. Vamos utilizar sua conta do GitHub para acessar o Codespaces. Caso não tenha uma conta, crie uma em [github.com](https://github.com).
2. Primeiro, faça o fork do repositório que contém os exercícios da disciplina. Acesse o link [FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh](https://github.com/vamperst/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh).
3. No canto superior da tela haverá o botão `Fork` para você copiar o repositório para sua conta do GitHub. Clique nele para copiar o repositório para sua conta.

![](img/fork1-1.png)

4. Após clicar no botão, você será redirecionado para a tela de fork do repositório. Deixe a opção `Copy the master branch only` desmarcada como no print para que todas as branches do repositório sejam copiadas. Clique em `Create Fork`.

![](img/fork2-1.png)

5. Agora vamos criar o Codespaces. Acesse o link [GitHub Codespaces](https://github.com/codespaces) e clique em `Get Started for free`.

![](img/codespaces1.png)

6. Clique em `New codespace` no canto superior direito para criar um novo ambiente.

![](img/codespaces2.png)

7. Deixe as opções da tela da seguinte forma e clique em `Create Codespace`:
   1. repository: `FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh`
   2. Branch: `master`
   3. Dev container configuration: `FIAP Lab`
   4. Region: `US East`
   5. Machine type: `2-core`

![](img/codespaces3.png)

8. Após a criação do ambiente, você será redirecionado para o Codespaces. Se quiser acompanhar a criação do ambiente, clique em `Building codespace` para abrir os logs.

![](img/codespaces4.png)

9. Essa criação pode demorar alguns minutos. Ao final, você verá a tela do Codespaces com o repositório clonado e pronto para uso.

> [!TIP]
> Deixe a aba do Codespaces aberta enquanto executa os próximos passos.

---

## Parte 2 - Acessando a conta AWS Academy

### Resultado esperado desta parte

Ao final desta etapa, você terá uma sessão ativa no laboratório da AWS Academy e acesso à sua conta AWS.

1. Caso ainda não tenha conta no AWS Academy:
   1. Entre no seu email da FIAP pelo endereço [webmail.fiap.com.br](http://webmail.fiap.com.br/).
   2. Seu email é no formato `rm<SEU RM>@fiap.com.br`. Exemplo: se seu RM for `12345`, seu email será `rm12345@fiap.com.br`. A senha é a mesma de portais.
   3. Procure o email de convite do Academy e siga as instruções.
   4. Ao conseguir entrar na plataforma, aparecerá uma turma que começa com `AWS Academy Learner Lab`. Clique em `Enroll` para aceitar e acessar.

2. Para entrar em uma conta do Academy que já existe, acesse [awsacademy.com/LMS_Login](https://www.awsacademy.com/LMS_Login). Ao entrar, localize a turma que começa com `AWS Academy Learner Lab` e clique em `Enroll`.

3. Dentro da plataforma, clique em `Cursos` na lateral esquerda e depois no curso da disciplina atual.

![](img/academy1.png)

4. Dentro do curso, clique em `Módulos` na lateral esquerda.

![](img/academy2.png)

5. Clique em `Iniciar os laboratórios de aprendizagem da AWS Academy`.

![](img/academy3.png)

6. Se for seu primeiro acesso, aparecerão os contratos de termos e condições. Role até o final e aceite. Caso já tenha feito isso antes, siga para o próximo passo.

![](img/academy4.png)

7. Clique no link iniciando com `Academy-CUR` para acessar a conta AWS. Caso peça consentimento, clique em `I agree` e execute o passo novamente.

![](img/academy8.png)

8. Clique em `Start Lab` para iniciar uma sessão. Esse processo pode demorar alguns minutos. Cada sessão dura 4 horas.

![](img/academy5.png)

![](img/academy6.png)

9. Quando tudo estiver pronto, a bolinha ao lado do texto `AWS` no canto superior esquerdo ficará verde. Clique em `AWS` para abrir a conta em outra aba do navegador.

![](img/academy7.png)

> [!IMPORTANT]
> Se a bolinha ainda não estiver verde, aguarde. Só siga quando a conta AWS abrir corretamente.

---

## Parte 3 - Criando o bucket base no S3

### Resultado esperado desta parte

Ao final desta etapa, você terá criado o bucket que será usado ao longo do curso.

1. Abra o console da AWS no [serviço S3](https://us-east-1.console.aws.amazon.com/s3/home?region=us-east-1#).
2. Clique em `Criar bucket`.

![](img/s3CreateBucket.png)

3. Dê o nome `base-config-<SEU RM>` e clique em `Criar`.

![](img/createBucket.png)

> [!TIP]
> Esse bucket será usado novamente em outros laboratórios. Confirme com atenção o nome antes de continuar.

---

## Parte 4 - Configurando as credenciais AWS no Codespaces

### Resultado esperado desta parte

Ao final desta etapa, o comando `aws s3 ls` deverá funcionar dentro do Codespaces.

1. Volte para a aba do Codespaces.
2. Verifique se o terminal está aberto. Caso não esteja, clique em `Terminal` na parte inferior da tela.
3. Para abrir o arquivo de configuração das credenciais AWS, execute:

```bash
code ~/.aws/credentials
```

4. Por enquanto o arquivo estará vazio. Na aba da AWS Academy, no canto superior direito, clique em `AWS Details` e depois em `Show` na seção `AWS CLI`.

![](img/codespaces6.png)

5. Copie o conteúdo da credencial para a área de transferência.

![](img/codespaces7.png)

6. Volte para o Codespaces, cole o conteúdo copiado no arquivo `~/.aws/credentials` e salve com `Ctrl+S`.

![](img/codespaces8.png)

7. Para validar, execute o comando abaixo no terminal:

```bash
aws s3 ls
```

Se tudo estiver correto, você verá a lista de buckets da sua conta, incluindo o bucket que acabou de criar.

![](img/codespaces9.png)

> [!IMPORTANT]
> Essa é a principal validação do setup. Só siga para a próxima etapa se esse comando funcionar sem erro.

---

## Parte 5 - Configurando a chave SSH

### Resultado esperado desta parte

Ao final desta etapa, o arquivo `~/.ssh/vockey.pem` deverá existir com permissão correta.

1. No terminal do Codespaces, execute o comando abaixo para criar a pasta e abrir o arquivo da chave SSH:

```bash
mkdir -p /home/vscode/.ssh/
code ~/.ssh/vockey.pem
```

2. De volta à aba da AWS Academy, clique em `AWS Details`, expanda a seção `SSH Key`, clique em `Show` e copie a chave privada.

![](img/codespacess12.png)

![](img/codespacess13.png)

3. Volte para o Codespaces, cole o conteúdo copiado no arquivo `~/.ssh/vockey.pem` e salve.
4. Ajuste as permissões da chave executando:

```bash
chmod 400 ~/.ssh/vockey.pem
```

---

## Checklist final

Antes de encerrar, confirme se você já fez tudo abaixo:

- criou o fork do repositório
- criou o Codespaces
- iniciou a sessão no AWS Academy
- criou o bucket `base-config-<SEU RM>`
- configurou `~/.aws/credentials`
- validou com `aws s3 ls`
- configurou `~/.ssh/vockey.pem`
- executou `chmod 400 ~/.ssh/vockey.pem`

**Pronto. Seu ambiente está configurado e pronto para começar os laboratórios.**

> [!WARNING]
> O passo de copiar as credenciais para o Codespaces é necessário para executar os comandos da AWS. Caso você feche o Codespaces e abra novamente, mas ainda esteja dentro do período de validade da sessão do AWS Academy, as credenciais continuarão funcionando. Caso contrário, será necessário copiar as credenciais novamente.

> [!CAUTION]
> **SEMPRE DESLIGUE** o ambiente ao final de cada aula para não gerar custos extras nem consumir suas horas gratuitas no Codespaces. Para desligar, acesse [GitHub Codespaces](https://github.com/codespaces), clique nos 3 pontinhos ao lado do ambiente e depois em `Stop Codespace`.

![](img/codespaces10.png)
