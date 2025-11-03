## Pré-requisitos

Antes de implantar o projeto, certifique-se de ter os seguintes pré-requisitos:

1. Conta AWS com permissões para criar clusters e nós EKS.
2. Cluster Kubernetes (EKS).
3. Ferramenta de linha de comando `kubectl` instalada e configurada para interagir com seu cluster Kubernetes.
4. `certbot` instalado para gerar certificados SSL da Let's Encrypt.
5. Registros DNS configurados para os domínios usados na configuração do Nginx.

## Estrutura do Projeto

O diretório do projeto contém os seguintes arquivos:

- `deploy-all.sh`: Um script shell para aplicar todas as configurações Kubernetes em sequência.
- `postgres-deployment.yaml`: Configuração de implantação e serviço para PostgreSQL.
- `plik-deployment.yaml`: Configuração de implantação e serviço para Plik.
- `wiki-deployment.yaml`: Configuração de implantação e serviço para Wiki.js.
- `onetimesecret-deployment.yaml`: Configuração de implantação e serviço para One Time Secret.
- `ejbca-deployment.yaml`: Configuração de implantação e serviço para EJBCA.
- `nginx-configmap.yaml`: ConfigMap para configuração do Nginx.
- `nginx-deployment.yaml`: Configuração de implantação e serviço para Nginx.

## Guia de Implantação Passo a Passo

### 1. Configurar Kubernetes na AWS

#### 1.1. Criar um Cluster EKS

1. Acesse o [Console de Gerenciamento da AWS](https://aws.amazon.com/console/).
2. Navegue até o serviço EKS.
3. Crie um novo cluster e siga as etapas de configuração fornecidas pela AWS.
4. Uma vez que o cluster esteja criado, vá para a aba "Compute".

#### 1.2. Criar Nós de Trabalho

1. Na aba "Compute", crie 3 nós para fazer parte do seu cluster.
2. Certifique-se de que os grupos de segurança e as funções IAM estejam corretamente configurados para os nós.
3. Verifique se todos os nós estão no estado "Ready".

#### 1.3. Verificar o Load Balancer

1. Verifique o Load Balancer criado pelo EKS para garantir que está configurado corretamente.
2. Verifique os grupos de segurança associados ao Load Balancer para garantir o fluxo de tráfego adequado.

### 2. Instalar Ferramentas Necessárias

Instale as ferramentas de desenvolvimento necessárias e o Certbot:

```bash
sudo yum groupinstall "Development tools"
sudo yum install certbot
```

### 3. Configurar DNS no CloudNS

1. Acesse [CloudNS](https://www.cloudns.net/main/).
2. Crie uma nova zona DNS com o nome de domínio desejado.
3. Dentro da zona, crie 4 registros de host para cada serviço (One Time Secret, Plik, Wiki.js, EJBCA) com tipo A, apontando para o IP público do seu plano de controle Kubernetes.

### 4. Obter Certificados SSL

Use o Certbot para gerar certificados SSL para seus domínios:

```bash
sudo certbot certonly --manual --preferred-challenges=dns -d secret.yourdomain.com -d plik.yourdomain.com -d wiki.yourdomain.com -d cert.yourdomain.com
```

### 5. Preparar Segredos do Kubernetes para os Certificados SSL

Crie um diretório chamado `certs` na sua pasta do projeto e copie os certificados SSL para este diretório:

```bash
sudo su
cd /etc/letsencrypt/live
cd [yourdomain.com]
cp fullchain.pem /home/ec2-user/[your-folder]/certs
cp privkey.pem /home/ec2-user/[your-folder]/certs
chown ec2-user:ec2-user /home/ec2-user/[your-folder]/certs/fullchain.pem /home/ec2-user/[your-folder]/certs/privkey.pem
exit
```

Crie um segredo Kubernetes para armazenar os certificados SSL:

```bash
kubectl create secret generic [yourname]secret --from-file=fullchain.pem=/home/ec2-user/[your-folder]/certs/fullchain.pem --from-file=privkey.pem=/home/ec2-user/[your-folder]/certs/privkey.pem
```

### 6. Atualizar Registros DNS

1. Acesse [CloudNS](https://www.cloudns.net/main/).
2. Na mesma zona DNS, exclua os registros A existentes.
3. Crie novos registros CNAME para os mesmos nomes de host, apontando para o LoadBalancer.

### 7. Implantar Serviços

Execute o script de implantação para aplicar todas as configurações Kubernetes:

```bash
sh deploy-all.sh
```

### 8. Verificar Implantações

Verifique se todos os pods estão funcionando corretamente:

```bash
kubectl get all
```

## Conclusão

Esta documentação fornece um guia abrangente para implantar várias aplicações em um cluster Kubernetes usando configurações YAML. Siga os passos cuidadosamente para garantir uma implantação bem-sucedida. Para quaisquer problemas ou melhorias, sinta-se à vontade para abrir uma issue ou contribuir para o repositório.

---

Por favor, substitua [yourdomain.com] e [your-folder] pelos valores reais correspondentes ao seu domínio e diretório do projeto. Certifique-se de seguir as instruções na ordem correta para garantir um processo de implementação suave.

---

