# 01.1 - Início de toda aula

Toda aula começa com o mesmo procedimento.

Você pode pular esta parte caso já tenha feito isso **na aula de hoje**.

> [!IMPORTANT]
> Durante o setup inicial você criou uma conta AWS Academy e um Codespaces. Caso ainda não tenha feito isso, siga o [tutorial de setup](./README.md) antes de continuar.

## Objetivo

Ao final deste roteiro, o seu Codespaces estará sincronizado com o repositório remoto e com as credenciais AWS atualizadas para a aula.

---

## Passo 1 - Atualizar o fork no GitHub

> [!TIP]
> Se o professor não pediu para atualizar o código do repositório, pule para o passo 3.

1. Acesse o repositório do seu fork da disciplina `FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh` no GitHub.
2. Clique em `Sync fork`. Caso exista algo para sincronizar, clique em `Update branch`.

![](img/sync1.png)

![](img/sync2.png)

> [!NOTE]
> Se não houver nada para sincronizar, a mensagem será `This branch is not behind the upstream`. Nesse caso, siga para o próximo passo.

![](img/sync3.png)

---

## Passo 2 - Abrir o Codespaces

3. Acesse [GitHub Codespaces](https://github.com/codespaces).
4. Clique no nome do Codespaces que você criou para as aulas.

![](img/codespacess11.png)

5. No terminal do Codespaces, atualize o repositório local com o conteúdo mais recente do remoto:

```bash
git pull origin master
```

---

## Passo 3 - Atualizar as credenciais AWS no Codespaces

6. No terminal do Codespaces, abra o arquivo de credenciais:

```bash
code ~/.aws/credentials
```

7. Com o arquivo aberto, vá para o [AWS Academy](https://www.awsacademy.com/vforcesite/LMS_Login) e entre no laboratório informado pelo professor.

![](img/ac1.png)

8. Na lateral esquerda, clique em `AWS Academy Learner Lab` e depois em `Módulos`.

![](img/ac2.png)

9. Clique em `Iniciar os laboratórios de aprendizagem da AWS Academy`.

![](img/ac3.png)

10. Clique em `Start Lab`. Aguarde até que a bolinha ao lado do texto `AWS`, no canto superior esquerdo, fique verde. Em seguida, clique em `AWS` para abrir a conta AWS em outra aba.

![](img/ac4.png)

11. Ainda na aba do AWS Academy, clique em `AWS Details`.

![](img/ac5.png)

12. Em `AWS CLI`, clique em `Show` para visualizar as credenciais de acesso.

![](img/ac6.png)

13. Copie as credenciais e cole no arquivo `credentials` que você abriu no passo 6. Depois, salve o arquivo e feche.

![](img/ac7.png)

---

## Passo 4 - Validar se está tudo certo

14. Para testar, execute o comando abaixo no terminal do Codespaces:

```bash
aws s3 ls
```

Se tudo estiver correto, você verá a lista de buckets da conta AWS.

## Resultado esperado

Ao final deste processo:

- o seu fork estará sincronizado, se necessário
- o repositório local no Codespaces estará atualizado
- as credenciais AWS do dia estarão configuradas
- o comando `aws s3 ls` estará funcionando

**Pronto. Seu ambiente está configurado e pronto para começar os laboratórios.**
