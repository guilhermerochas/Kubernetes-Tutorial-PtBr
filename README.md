# Before you Start

This is a translation to portuguese of an [Article](https://www.freecodecamp.org/news/learn-kubernetes-in-under-3-hours-a-detailed-guide-to-orchestrating-containers-114ff420e882/) created by Rinor Maloku about Kubernetes, all the material provided, commands, image sources are all of his authorship just shared for the ones who strugle with the english language.

## Links
- Maloku's github profile: https://github.com/rinormaloku
- Project Repository: https://github.com/rinormaloku/k8s-mastery
---

# Aprenda Kubernetes em 3 Horas: Um Guia detalhado de Orquestração de Containers

Por que bancos estão me pagando tanto dinheiro para algo tão simples quanto Kubernetes? quando qualquer -- qualquer pessoa pode aprender dentro de três horas?

se voce duvida, eu te desafio a tentar! ao final desse artigo você será capaz de rodar aplicações baseadas em microserviços em um Cluster de Kubernetes. E eu garanto isso por que essa é a maneira que eu introduzo nossos clientes ao Kubernetes 

*Mas o que esse guia tem de diferente de qualquer outro, Rinor?*

Muita coisa! a maioria dos guias começa simples: Conceitos de Kubernetes e comandos do Kubectl. Esses guias acham que o leitor conhece sobre desenvolvimento de aplicações, microserviços e containers Docker.

Nesse artigo vamos de:

1. Rodar uma aplicação baseada em microserviço no seu computador
2. Fazer imagens Docker para cada serviço dos microserviços da aplicação
3. Introdução à Kubernetes. Deploy da aplicação de microserviçõs dentro do Cluster administrado pelo Kubernetes

O processo gradual provem a amplitude para os meros mortais entenderem a simpliciade do Kubernetes. Sim, Kubernetes é simples quando se entende o contexto que é usado. Sem mais enrolação vamos ver o que vamos fazer.

## Demo da Aplicação

A aplicação tem uma função. Ela pega uma frase como entrada de dados. Usando Analise de Texto, ela calcula a emoção da frase.

![Aplicação de Analise Sentimental](https://cdn-media-1.freecodecamp.org/images/Rl5B3SRE5U5IiIM-8-1HnZdnwMx1TzegzV3D)

Img. 1.Aplicação Web de Analise Sentimental


De uma perspectiva técnica, a aplicação consiste em três microserviços. Cada um com sua funcionalidade especifica:

- **SA-Frontend**: Um servidor web Nging que **server para nós arquivos estaticos em ReactJS**
- **SA-WebApp**: Uma aplicação Web em Java que **lida com requisições** do nosso frontend
- **SA-Logic**: Uma aplicação Python que **realiza Analise Sentimental**

É importante saber que Microserviços não vivem isoladamente, eles disponibilizam "separação de preocupações (separation of concerns)" mas ainda tem que interagir um com o outro.

![Fluxo de Dados na Aplicação Web de Analise Sentimental](https://cdn-media-1.freecodecamp.org/images/JwIBwPsTfBmelKgSrCCkEZuTzC5Ty1pZi3K7)

Img. 2. Fluxo de Dados na Aplicação Web de Analise Sentimental


Essa interação é melhor ilustrada mostrando como os dados fluem entre eles:

1. A aplicação client requisita o index.html (o qual traz scripts empacotados na aplicação ReactJS)
2. O usuario interage com a aplicação acionando a requisição para a WebApp Spring)
3. A Aplicação Web Spring dirige a requisição para a analise sentimental da aplicação Python.
4. A aplicação Python calcula o sentimento e retorna uma resposta como resultado.
5. A Aplicação Web Spring retorna a resposta para o app React. (o qual apresenta a informação para o usuario.)

O codigo para todas essas aplicações pode ser encontrado [nesse repositorio](https://github.com/rinormaloku/k8s-mastery). Eu recomendo clona-lo Imediatamente por que nós vamos construir coisas incriveis juntos.

## 1. Rodando a aplicação baseada em Microserviços no seu computador

Precisamos rodar todos os três serviços. Vamos começar com o mais atrativo, a aplicação front-end.

### Configurando React para Desenvolvimento Local

Para rodar a aplicação React vamos precisar de NodeJS e NPM instalados no computador. Depois de intala-los navegue no seu terminal para o diretório **sa-frontend** e escreva o seguinte comando:

```bash
  npm install
```

Ele baixa todas as dependencias Javascript da aplicação React e coloca elas numa pasta **node_modules** (Dependencias são definidas no arquivo package.json). Depois que todas as dependencias são solucionadas execute o comando seguinte:

```bash
  npm start
```

E é isso! nós rodamos nossa aplicação react e por padrão você pode acessá-la em **localhost:3000**. Sintase livre para modificar o código e ver seus resultados imediatamente no navegador. Isso é possivel através do **Hot Module Replacement**. Fazendo com que o desenvolvimento do nosso front end seja uma delicinha!

### Fazendo nosso App React pronto para produção

Para produção nós precisamos tranformar nossa aplicação em arquivos estáticos e servi-los a um servidor web.

Para tranformar a aplicação react navegue no seu terminal para o diretório **sa-frontend**. E execute o comando a seguir:

```bash
  npm run build
```

Ele gera uma pasta chamada **build** na estrutura do projeto. Essa pasta contem os arquivos estáticos necessarios para nossa aplicação ReactJS.

### Servindo arquivos estáticos com Nginx

Instale e rode o Servidor Web Nginx ([veja como](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)). E então mova o conteudo de sa-frontend/build para a pasta \[seu_diretorio_de_instalacao_nginx\]/**html**.

Dessa forma o arquivo gerado index.html vai ser acessivel em \[seu_diretorio_de_instalacao_nginx\]/html/index.html. **Esse é o arquivo padrão que o Nginx serve**.

Por padrão o Servidor Web Nginx escuta na porta 80. Você pode especificar uma porta diferente atualizando a propriedade server.listen no arquivo \[seu_diretorio_de_instalacao_nginx\]/conf/nginx.conf.

Abra o seu navegador e va até o endpoint localhost:80, veja a aplicação ReactJS aparecer.

![Aplicação React servida pelo Nginx](https://cdn-media-1.freecodecamp.org/images/EOcacd0QABnXiFAXVHPpWcD9scHzvr7jq0Fp)

Img. 3.Aplicação React servida pelo Nginx

### Inspecionando o Código

No arquivo **App.js** nós podemos ver que ao pressionar o botão Send ele aciona o método analyzeSentence. O código para esse método é mostrado abaixo. (Cada linha comentada com #Numero será explicada abaixo do script):

```javascript
analyzeSentence() {
    fetch('http://localhost:8080/sentiment', {  // #1
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
                       sentence: this.textField.getValue()})// #2
    })
        .then(response => response.json())
        .then(data => this.setState(data));  // #3
}
```

#1: URL na qual uma chamada POST é feita. (Uma aplicação deveria escutar as chamadas nessa URL).

2#: O corpo da Requisição manda para a aplicação o conteudo mostrado abaixo:
```javascript
{
  sentence: “I like yogobella!”
} 
```

#3: A resposta atualiza o estado do componente. Isso aciona uma re-renderização do componente. se nós recebermos dados, (ex. o objeto JSON contendo a frase digitada e sua polaridade) nós iriamos mostrar o componente polarityComponent por que a condição seria satisfeita e iriamos definir o componente:

```javascript
const polarityComponent = this.state.polarity !== undefined ?
    <Polarity sentence={this.state.sentence} 
              polarity={this.state.polarity}/> :
    null;
```
Tudo parece correto. Mas o que estamos esquecendo? se voce disse que não configuramos nada para escutar na localhost:8080, você está certo! Nós devemos iniciar nossa Aplicação Spring Web para escutar nessa porta!

![Microserviço da Aplicação Web Spring que estamos esquecendo](https://cdn-media-1.freecodecamp.org/images/KNFf142A66wPteChS7IQmcZA8ohQTZRA8U7E)

Img. 4.Microserviço da Aplicação Web Spring que estamos esquecendo

### Configurando a Aplicação Web Spring

Para iniciar a aplicação Spring precisamos ter o JDK8 e o Maven instalados. (suas variáveis de ambiente precisam ser configuradas também). Depois da instalação você pode continuar para a parte seguinte.

### Empacotando a Aplicação em um Jar
Navegue no seu Terminal para o diretório **sa-webapp** e digite o comando a seguir:

```bash
  mvn install
```

Isso vai gerar uma pasta chamada **target**, no diretório **sa-webapp**. na pasta **target** nos temos nossa aplicação Java empacotada como um jar: ‘**sentiment-analysis-web-0.0.1-SNAPSHOT.jar**’

### Iniciando nossa Aplicação Java

Navegue até o diretorio target e inicie a aplicação o comando seguinte:

```bash
  java -jar sentiment-analysis-web-0.0.1-SNAPSHOT.jar
```
Droga... tivemos um erro. Nossa aplicação falhou ao executar e nossa unica pista é a excessão está no stack trace:

```bash 
  Error creating bean with name 'sentimentController': Injection of autowired dependencies failed; nested exception is java.lang.IllegalArgumentException: Could not resolve placeholder 'sa.logic.api.url' in value "${sa.logic.api.url}"
```
A informação importante está aqui no placeholder sa.logic.api.url no **SentimentController**. Vamos dar uma olhada nisso!

### Analisando o Código

```java
@CrossOrigin(origins = "*")
@RestController
public class SentimentController {

    @Value("${sa.logic.api.url}")    // #1
    private String saLogicApiUrl;
    
    @PostMapping("/sentiment")
    public SentimentDto sentimentAnalysis(
        @RequestBody SentenceDto sentenceDto) 
    {
        RestTemplate restTemplate = new RestTemplate();
        
        return restTemplate.postForEntity(
                saLogicApiUrl + "/analyse/sentiment",    // #2
                sentenceDto, SentimentDto.class)
                .getBody();
    }
}
```

1. O **SentimentController** tem um campo definido pela propriedade `sa.logic.api.url`.
2. A String saLogicApiUrl é concatenada com o valor "/analyse/sentiment". Juntos eles formam a URL para fazer a requisição pela Analise Sentimental.

#### Definindo a Propriedade

no Spring a fonte padrão de propriedades é **aplication.properties**. (Localizada em *sa-webapp/src/main/resources*). Mas essa não é a unica maneira de definir a propriedade, ela pode ser passara com o comando anterior tambem:

```bash
  java -jar sentiment-analysis-web-0.0.1-SNAPSHOT.jar --sa.logic.api.url=OQUE.FOR.A.SA.LOGIC.API.URL
```

A propriedade deve ser inicializada com o valor que define onde nossa aplicação Python está rodando, dessa maneira vai nos permirit que nossa Aplicação Spring Web saiba onde encaminhar as mensagens no tempo de execução.

Para simplificar as coisas vamos decidir que iremos rodar nossa aplicação python em `localhost:5000.` Só não vamos esquecer!

Rode o comando abaixo e iremos estar prontos para ir para o ultimo serviço: a aplicação python.

![](https://cdn-media-1.freecodecamp.org/images/gRyXOa3fibWNB7s1DJiu31nB0ziy38FjCWe5)

### Configurando a Aplicação Python

Para inciar a aplicação Python, iremos precisar de Python3 e Pip instalados. (Suas variáveis de ambiente precisam ser configuradas também).

### Instalando Dependencias

Navegue no terminal para o diretório **sa-logic/sa** ([repositório](https://github.com/rinormaloku/k8s-mastery)) e digite o comando a seguir:

```bash
  python -m pip install -r requirements.txt
  python -m textblob.download_corpora
```

### Inicializando o App

Depois de usar o Pip para instalar as dependencias nós estamos prontos para inciciar nossa aplicação executando o comando a seguir:

```bash
  python sentiment_analysis.py
  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

Isso significa que nossa aplicação está rodando e escutando Requisições HTTP na porta 5000 em localhost.

### Analisando o Código

Vamos investigar o codigo para entender o que está acontecendo em nossa aplicação python **Sa Logic**.

```python
from textblob import TextBlob
from flask import Flask, request, jsonify

app = Flask(__name__)                                   #1

@app.route("/analyse/sentiment", methods=['POST'])      #2
def analyse_sentiment():
    sentence = request.get_json()['sentence']           #3
    polarity = TextBlob(sentence).sentences[0].polarity #4
    return jsonify(                                     #5
        sentence=sentence,
        polarity=polarity
    )
    
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)                #6
```

1. Instancia o objeto Flask.
2. Define o caminho no qual a requisição POST pode ser feita.
3. Extrai a propriedade "sentence" do corpo da requisição.
4. Inicializa um objeto anonimo TextBlob e pega a polaridade da primera frase. (Só temos apenas uma)
5. Retorna a resposta com o corpo contendo a frase e a polaridade para quem a chamou.
6. Roda o **app** objeto flask para escutar por requisições em 0.0.0.0:5000 (chamando para localhost:5000 irá alcançar a aplicação também).

**Os serviços estão setados para se comunicar um com o outro. Reabra o frontend em localhost:80 e faça uma tentativa antes de continuar!**

![Arquitetura completa de Microserviços](https://cdn-media-1.freecodecamp.org/images/Wfr68VDVe8eOlB0Z9sM8sunj60L7UZD1Hu9v)

Img. 6. Arquitetura completa de Microserviços

Na proxima sessão, iremos examinar como iniciar os serviços em Containers Docker, já que é um pré-requisito para ser capaz de rodá-los em um Cluster Kubernetes.

## 2. Construindo imagens de containeres para cada serviço

Kubernetes é um orquestrador de containers. Entendido isso nós precisamos de containers para poder orquestra-los. Mas o que são containers? A melhor resposta está na documentação do docker.

> Uma imagem de container é um pacote de um pedaço de software leve, único e executavel que inclui tudo necessario para rodalo: código, tempo de execução, ferramentas de sistema, bibliotecas de sistema, configurações. Disponivel para aplicações baseadas em Linux e Windows, software containerizado vai sempre rodar o mesmo, não importando o ambiente.

Isso significa que containers rodam em qualquer computador -- mesmo em servidores de produção -- **sem nenhuma diferença**

### Servindo arquivos estáticos em React de uma VM

Os contras de usar uma Maquina Virtual:

1. Ineficiente em recursos, cada VM é sobrecarregada de um SO inteiro.
2. É dependente de plataforma. O que funcionou no seu computador pode não funcionar num servidor de produção
3. Pesada e lerda na escalabilidade comparada com Containeres.

![Servidor Web Nginx com arquivos estáticos em uma VM](https://cdn-media-1.freecodecamp.org/images/vP3JZyOygXDzTh7I650wZHtHWkv56ioduUJS)

Img. 7.Servidor Web Nginx com arquivos estáticos em uma VM

### Servindo arquivos estáticos em React de um Container

Os pros de usar um Container:

1. Eficiente em recursos, usa o SO Host com a ajuda do Docker.
2. Independente de plataforma. O container que você rodou no seu computador irá rodar em qualquer lugar.
3. Leve usando camada de imagens

![Servidor Web Nginx com arquivos estáticos em um Container](https://cdn-media-1.freecodecamp.org/images/6I9ZEnnQNMqeTCK8kRWoOjDucfLCJqjAUGWd)

Img 8.Servidor Web Nginx com arquivos estáticos em um Container

### Criando uma imagem de um container para o App React (Introdução ao Docker)

O fundamento basico para um container Docker é a .dockerfile. A **Dockerfile** começa com um container de base e continua com uma sequencia de instruções de como criar uma nova imagem de container que satisfaz as necessidades de sua aplicação.

Antes de começar definindo a Dockerfile, vamos lembrar que os passos que tomamos para servir os arquivos estáticos usando nginx:

1. Tranformar em arquivos estáticos (npm run build)
2. Iniciar o servidor nginx
3. copiar o conteudo para a pasta **build** do seu projeto sa-frontend para nginx/html.

Na proxima sessão, você ira notar paralelos em como criar Containers é similar no que fizemos em nossa configuração React local.

### Definindo a Dockerfile para SA-Frontend

A instrução no Dockerfile para o SA-Frontend é apenas uma tarefa de dois passos. Isso é por que o Time da Nginx nos disponibilizou [uma imagem de base](https://hub.docker.com/_/nginx/) para Nginx, a qual nós usamos para trabalhar encima dela. Os dois passos são:

1. Inicie da **Imagem de base Nginx**
2. Copiar o diretório **sa-frontend/build** para o diretório nginx/html do container.

A maneira que parece convertida em uma Dockerfile:
```dockerfile
FROM nginx
COPY build /usr/share/nginx/html
```

Isso não é incrivel? é até humanamente legivel, vamos recapitular:

Inicie de uma imagem nginx. (Qualquer coisa que tenham feito aqui). Copiar o diretorio **build** para o diretório **nginx/html** na imagem. É isso!

Você talvez esteja se perguntando, como que eu sabia para onde copiar os arquivos da build? ex. `/usr/share/nginx/html`. Muito simples: estava documentada na [imagem](https://hub.docker.com/_/nginx/) nginx no Docker Hub.

### Criando e Fazendo *Push* de containers

Antes de nos podemos realizar *push* de nossas imagens, precisamos realizar um Container Registry (Registro de Container) para realizar o host de nossas imagens. Docker Hub é um serviço livre de containers na nuvem que nós podemos usar de demonstração. Você precisa realizar essas três tarefas antes de continuar:

1. [Instalar Docker CE](https://www.docker.com/products/container-runtime)
2. Registrar-se no Docker Hub.
3. Logar ao executar o comando abaixo:

```bash
docker login -u="$DOCKER_USERNAME" -p="$DOCKER_PASSWORD"
```
Depois de completar as tarefas acima navegue para o diretorio **sa-frontend**. Então execute o comando abaixo (substituindo $DOCKER_USERNAME com seu nome de usuario do docker hub. Por exemplo rinormaloku/sentiment-analysis-frontend)

```bash
docker build -f Dockerfile -t $DOCKER_USER_ID/sentiment-analysis-frontend .
```

Podemos remover `-f Dockefile` por que nós já estamos no diretório contendo o Dockerfile.

Para fazer o *push* de imagens, use o comando docker push:

```bash
docker push $DOCKER_USER_ID/sentiment-analysis-frontend
```

Verifique no seu repositório do docker hub que sua imagem teve um *push* realizado com sucesso.

### Rodando o container

Agora a imagem em `$DOCKER_USER_ID/sentiment-analysis-frontend` pode ser baixada por qualquer um:

```bash
docker pull $DOCKER_USER_ID/sentiment-analysis-frontend
docker run -d -p 80:80 $DOCKER_USER_ID/sentiment-analysis-frontend
```

Nosso container Docker está rodando!

Antes de continuarmos vamos elaborar o 80:80 que eu acho confuso:
- O primeiro 80 é a porta do host (ex. meu computador)
- o segundo 80 é a porta do container nas quais as chamadas devem ser encaminhadas.

![Mapeamneto de porta do Host para o Container](https://cdn-media-1.freecodecamp.org/images/uUv5pZc6QErqJVcacC0vU-QEvjDjVF1VlQ9l)

Img. 9. Mapeamneto de porta do Host para o Container

Ele mapeia da <PortaUsuario> para a <PortaContainer>. Significa que chamadas da porta 80 do host devem ser mapedas para a porta 80 do container, como mostrado na figura 9.
  
Por conta que a porta foi rodada no host (seu computador) ela deve ser acessivel em localhost:80. Se voce não tiver suporte do docker nativo, voce pode abrir em <ip da docker-machine>:80. Para descobrir o ip da sua docker-machine execute `docker machine ip`.

De uma tentativa! Voce deve ser capaz de acessar a aplicação react no endpoint.

### O Dockerignore

Nós vimos antes que construir imagens para SA-Frontend foi devagar, perdoe, **Muito devagar**. Isso foi por que o **build context** que foi mandado para o Docker Deamon. Em mais detalhes, o diretório **build context** é definido pelo ultimo argumento no comando de build (O seguinte ponto), no qual especifica o contexto de construção. E no nosso caso, ele inclui as seguintes pastas:

```bash
  sa-frontend:
  |   .dockerignore
  |   Dockerfile
  |   package.json
  |   README.md
  +---build
  +---node_modules
  +---public
  \---src
```
Mas os unicos dados que precisamos é a pasta **build**. Enviar qualquer outra coisa é perda de tempo. Nós podemos melhorar nosso tempo de build nos desfazendo dos outros diretórios. Ai que entra o `.dockerignore`. Para voce que já esta familiarizado com `.gitignore`, ex. adicione todos os diretórios que quer ignorar no arquivo `.dockerignore`, como é mostrado abaixo:

```bash
  node_modules
  src
  public
```

O arquivo `.dockerignore` deve estar na mesma pasta que o Dockerfile. Agora construir as imagens irá demorar apenas segundos.

Vamos continuar com a Aplicação Java

### Construindo a imagem do container para a Aplicação Java

Adivinha! Você aprendeu quase tudo sobre criar imagens de containers! esse é o motivo de por que essa parte é estremamente curta.

Abra a Dockerfile em **sa-webapp**, e você irá encontrar duas novas palavras chaves:

```dockerfile
ENV SA_LOGIC_API_URL http://localhost:5000
…
EXPOSE 8080
```

A palavra chave **ENV** declara uma Variavel de Ambiente dentro do container docker. Isso vai nos habilitar à prover uma URL para a API de Analise Sentimental quando começar o Container.

Adicionalmente, a palavra chave **EXPOSE** expõe a porta que nós queremos acessar depois. **Mas olha!!!** Nós não fizemos isso na Dockerfile do SA-Frontend, belo ponto! Isso é apenas para fins de documentação, em outras palavras isso server de informação para a pessoa que está lendo a DockerFile.

Você deve estar familiarizado com a construção e o *push* de imagens de container. Se tiver quaisquer dificuldade leia o arquivo README.md no diretório **sa-webapp**.

### Construindo a imagem do container para a Aplicação Python

No Dockerfile em **sa-logic** não tem nenhuma palavra chave nova. Agora você pode se chamar um Mestre Docker?

Para a construção e o *push* de imagens de container leia o README.md no diretório **sa-logic**

### Testando a Aplicação Containerizada

Voce confia em algo que nem testou? Nem eu. Vamos dar uma testada nos containers.

1. Rode o container **sa-logic** e o configure para escutar na porta 5050:

```bash
  docker run -d -p 5050:5000 $DOCKER_USER_ID/sentiment-analysis-logic
```
2. rode o container **sa-webapp** e o configure para escutar na porta 8080, adicionamente nos precisamos mudar a porta na qual a nossa aplicação ira escutar por sobreescrever nssa variabel de ambiente SA_LOGIC_API_URL.

```bash
   $ docker run -d -p 8080:8080 -e SA_LOGIC_API_URL='http://<id_do_container ou ip da docker-machine>:5000' $DOCKER_USER_ID/sentiment-analysis-web-app
```

De uma olhadada no [README](https://github.com/rinormaloku/k8s-mastery/blob/master/sa-webapp/README.md) em como conseguir o ip do contaner ou da docker-machine.

3. Rode o container **sa-frontend**:

```bash
  docker run -d -p 80:80 $DOCKER_USER_ID/sentiment-analysis-frontend
```
Estamos prontos. Abra o navegador em **localhost:80.**

**Atenção:** Se você mudar a porta para a sa-webapp, ou se estuver usando o ip da docker-machine, você vai precisar atualizar o App.js no **sa-frontend** no método analyzeSentence para buscar os dados em uma nova Porta ou IP. Depois voce precisa dar um *build*, e usar a imagem atualizada.

![Microserviços rodando em Containers](https://cdn-media-1.freecodecamp.org/images/gdDm95hkRv-AnNmuHUFDIONucxEWcvXN1p34)

Img. 10.Microserviços rodando em Containers

### Teaser Cerebral -- Por que Kubernetes?

Nessa sessão, nós aprendemos sobre as Dockerfiles, como usalas para construir imagens, e os comando para fazer *push* para a Docker Registry. Adicionalmente, investigamos como reduzir o numero de arquivos mandados para um *build context* ao ignorar arquivos inuteis. E no final, nos tivemos nossa aplicação rodando em containers. Então por que Kubernetes? Vamos investigar mais a fundo no proximo arquivo, mas eu quero deixar um teaser cerebral para você.

- Nossa aplicação web de Analise Sentimental se torna um hit de sucesso no mundo e temos milhoes de requisições diarias por minuto para analizar e temos uma enorme carga em **sa-webapp** e **sa-logic**. Como podemos escalar os containers?

### Introdução ao Kubernetes

Eu te prometo e não estou exagerando que ao final desse artigo você estará se pergundando "Por que não chamamos isso de Supernetes?".

![Supernetes](https://cdn-media-1.freecodecamp.org/images/6z5-sOpVzRF1YeB2kQzrXakp2kBiGDBlMx4t)

Img. 11.Supernetes

Se você seguiu esse artigo desde o começo nós cobrimos muita consistencia e muito conhecimento. Você deve estar preocupado que essa vai ser a parte mais dificil, mas, essa é a mais simples. O motivo por que aprender Kubernetes é assustador é por conta de "todo o resto" que nós cobrimos tambem.

### O qué é Kubernetes

Depois de que começamos nossos microserviços de containers nós tivemos uma pergunta, vamos elaborar melhor em um formato de Perguntas e Respostas:  
P: Como escalamos containers?  
R: Nós subimos mais um.  
P: Como balanceamos carga entre eles? E se o servidor já estiver usando o maximo e nós precisemos de outro servidor para nosso container? como calculamos a melhor utilização de hardware?  
R: Éééé... hmmmm... (xô ve na internet).  
P: Fazer atualizações sem quebrar nada? e se quebrar, como podemos voltar pra versão que está funcionando.

Kubernetes soluciona todas essas perguntas (e mais!). Minha tentativa de reduzir Kubernetes em apenas umas frase seria: "Kubernetes é uma Orquestração de Containers, que abstrai toda a infraestrutura interna. (Onde os containers rodam)".

Temos uma ideia vaga sobre Orquestra de Containers. Nós vamos ver em pratica na continuação desse artigo, mas essa é a primeira vez que estamos lendo sobre "abstrai a infraestrutura interna". vamos dar uma olhada nisso.

### Abstraindo a infraestrutura interna.

Kubernetes abstrai a infraestrutura interna provendo uma API simples na qual podemos mandar requisições. Essas requisições avisam o Kubernetes para os encontrar nas melhores das capacidades. Por exemplo, É tão simples quanto pedir "Kubernetes suba 4 containers da imagem x". Então Kubernets vai achar os nós pouco utilizados nos quais ele irá rodar os novos containers (veja Img. 12.).

![Requisição para o Servidor da API](https://cdn-media-1.freecodecamp.org/images/oRhjNBu9XyT74V6dxJVs1YJhgoC2eMU8TsCX)

Img. 12.Requisição para o Servidor da API

O que isso significa para o desenvolvedor? que ele não tem que se importar para o numero de nós, onde eles começam e como eles se comunicam. Ele não tem que se preocupar com a otimização de hardware ou se preocupar com nós caindo (E pela *Lei de Murphy* eles vão cair), por que novos nós podem ser adicionados para o Cluster de Kubernetes. Nesse periodo de tempo Kubernetes vai rodar os containers nos outros nós que ainda estão rodando. Fazendo uma das melhores capacidades possivel.

Na figura 12 nos podemos ver algumas coisas novas:

- **API Server (Servidor da API)**: Nossa unica maneira de interagir com o Cluster. Seja iniciar ou parar outro container (ééé pods\*)
- **Kubelet**: monitora os containers (ééé pods\*) dentro do nó e comunica com o nó *master*.
- **Pods\***: Inicialmente só pense que pods são containers.

E vamos parar por aqui, por que se aprofundar vai fazer com que perdamos nosso focuo e podemos fazer isso depois, existe varios recursos onde você pode aprender, como a documentação oficial (a maneira dificil) ou lendo o livro incrivel [Kubernetes in Action](https://www.amazon.com/Kubernetes-Action-Marko-Luksa/dp/1617293725), por [Marko Lukša](https://twitter.com/markoluksa).

### Padronizando os Provedores de Serviços de Nuvem

Outro forte ponto que Kubernetes leva consigo, é que ele padroniza os Provedores de Serviços de Nuvem (PSNs). Essa é uma frase forte, mas vamos elaborar com um exemplo:

&mdash; Um especialista em Azure, Google Cloud Platform ou outro PSN acaba trabalhando em um projeto em uma nova PSN, na qual ele não tem nenhuma experiencia com ela. Pode haver varias consequencias, vou falar algumas: Ele pode perder o prazo; A empresa pode querer contratar mais recursos, e muito mais.

Em contraste, com Kubernetes nem temos esse problema. Por que você seria capaz de executar os mesmos comandos para a API Server não importando o PSN. Você em uma maneira de requisitar declarativamente da API Server **o que quiser**. Kubernetes abstrai e implementa o **como** para a PSN em questão.

Dareium segundo para entender -- Essa é um aspecto extremamente poderoso. Para empresas isso significa que elas não estão presas ao PSN, e elas podem continuar. Eles ainda vão ter a esperteza, ainda vão ter os recursos, e podem fazer isso *mais barato*!

Tudo dito, vamos na proxima sessão botar Kubernetes na Pratica.

### Kubernetes na Pratica -- Pods

Nos configuramos os microserviços para rodar em containers e foi um processo pesado, mas funcionou. Tambem mencionamos que essa não era uma solução escalavel e resiliente e que Kubernetes resolve esses problemas. Na continuação desse artigo, nos vamos migrar nossos serviços para um resultado final mostrado na imagem 13, onde os Containers estão orquestrados pelo Kubernetes.

![Microserviços rodando em um Cluster Gerido por Kubernetes](https://cdn-media-1.freecodecamp.org/images/mrA3VBYh2pbG7qH9wnsMj-QxRxZ2MAqA5oTt)

Img. 13.Microserviços rodando em um Cluster Gerido por Kubernetes

Nesse artigo iremos usar Minikube para debugar localmente, mesmo que tudo o que será apresentado funciona tambem no Azure e no Google Cloud Platform.

### Instalando e Iniciando Minikube

Siga a documentação oficial para a instalação do [Minikube](https://minikube.sigs.k8s.io/docs/start/). Durante a instalação do Minikube, você tambem ira instalar **Kubectl**. Esse é o cliente que faz requisições ao Kubernetes API Server.

Para iniciar o Minikube execute o comando `minikube start` e depois de completo execute `kubectl get nodes` que você receberá a seguinte output

```bash
kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    11m       v1.9.0
```

Minikube prove para nos com um Cluster Kubernetes com um nó, mas lembre-se nos não nos importamos quantos nós existem, Kubernetes abstrai todos eles, e para nós aprendermos Kubernetes isso não tem importancia. Na proxima sessão, nos iremos começãr com nosso primeiro recurso de Kubernetes \[RUFEM OS TAMBORES\] **O Pod**.

### Pods

Eu amo containers, e até o momento você ama tambem. Então por que Kubernetes resolveu nos dar Pods como a menor unidade computacional implementavel? o que um pod faz? pods podem ser compostos de um ou um grupo de containers que dividem o mesmo ambiente de execução.

Mas nos realmente precisamos rodar dois containers em um pod? Ééé... Geralmente, você rodaria apenas um container mas é isso que vamos fazer nos nossos exemplos. Mas para casos onde por ex. dois containers precissem dividir volumes, ou precisem comunicar-se atravez de um processo interno ou caso contrario eles são extremamente acoplados, isso é feito possivel usando **Pods**, outra usabilidade que Pods nos permitem fazer é que eles não estão presos aos containers do Docker, se você quiser podemos usar outras tecnologias por ex. [Rtk](https://coreos.com/rkt/).

![Propriedades de Pods](https://cdn-media-1.freecodecamp.org/images/DiiFgshSEsYe9Rj2AHAUtJUI90CVH53VdioW)

Img. 14.Propriedades de Pods

Para resumir, as principais propriedades de Pods são (tambem mostrado na imagem 14):
1. Cada pod tem seu endereço de Ip unico no Cluster de Kubernetes.
2. Pods podem ter multiplos containers dividindo a mesma porta, assim como comunicação através do localhost (entendivelmente eles não podem se comunicar pela mesma porta), e comunicação com outros containers de outros pods deve ser feita na conjunção do ip do pod.
3. Containers num pod podem dividir o mesmo volume\*, mesmo ip, porta, namespace IPC

\*Containers tem seus sistemas de arquivos isolados, mesmo que eles sejam capazes de compartilhar dados usando o recurso Kubernetes de **Volumes**.

Isso é mais do que informação suficiente para nos continuarmos, mas para satisfazer sua curiosidade cheque a [documentação oficial](https://kubernetes.io/docs/concepts/workloads/pods/).

### Definição de Pod

Abaixo temos um arquivo de manifesto para nosso primeiro pod **sa-frontend**, e então temos abaixo uma explicação de todos os pontos.

```yaml
apiVersion: v1
kind: Pod                                            # 1
metadata:
  name: sa-frontend                                  # 2
spec:                                                # 3
  containers:
    - image: rinormaloku/sentiment-analysis-frontend # 4
      name: sa-frontend                              # 5
      ports:
        - containerPort: 80 
```

1. **Kind**: especifica o tipo do recurso de Kubernet que queremos criar. Nesse caso, um **Pod**.
2. **Name**: define o nome do nosso recurso. Nos nomeamos de **sa-frontend**.
3. **Spec** no objeto ele define o estado que desejamos para o recurso. O mais importante é a propriedade que todas as Spec dos Pods são Arrays de containers.
4. **Image** é a a imagem do container que queremos para inicializar esse pod.
5. **Name** é o nome unico para o container no pod
6. **Container Port**: a porta na qual o container vai escutar. isso é apenas uma indicação ao leitor (remover a porta não vai restringir o acesso).

### Criando o pod para SA Frontend

Você pode achar o arquivo da definição do pod acima em `resource-manifests/sa-frontend-pod.yaml.` Você pode tanto navegar pelo terminal para a pasta ou prover a localização completa na linha de comando. Então execute o comando:

```bash
kubectl create -f sa-frontend-pod.yaml
pod "sa-frontend" created
```

Para checar se o Pod está ropdando execute o comando seguinte:
```bash
kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
sa-frontend                   1/1       Running   0          7s
```
Se ele ainda estiver em **ContainerCreating** você pode executar o comando acima com o argumento `--watch` para atualizar a informação quando o pod estiver em um Running State.

### Acessando a aplicação externamente

Para acessar a aplicação externamente nos criamos um recueso em Kubernetes do tipo **Service**, que veremos no proximo artigo, o qual é a maneira genuina de implementação, mas para uma breve debugada nos temos uma outra opção, e ela é a port-forwarding:

```bash
kubectl port-forward sa-frontend 88:80
Forwarding from 127.0.0.1:88 -> 80
```

Abra seu navegador em **127.0.0.1:88** e você vai obter a aplicação react.

### A maneira errada de escalar

Nos dissemos que uma das principais caracteristicas de Kubernetes é a escalabilidade, para provar vamos rodar outro pod. Para faer isso vamos criar outro recurso de pod, com a seguinte definição

```yaml
apiVersion: v1
kind: Pod                                            
metadata:
  name: sa-frontend2      # A unica mudança
spec:                                                
  containers:
    - image: rinormaloku/sentiment-analysis-frontend 
      name: sa-frontend                              
      ports:
        - containerPort: 80
```

Crie um novo pod executando o seguinte comando:

```bash
kubectl create -f sa-frontend-pod2.yaml
pod "sa-frontend2" created
```
Verifique que o seguindo pod esta rodando executando:

```bash
kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
sa-frontend                   1/1       Running   0          7s
sa-frontend2                  1/1       Running   0          7s
```

Agora temos dois pods rodando!

**Atenção:** Essa não é a solução final, e tem diversas falhas. Vamos melhorar em outra sessão para outro recurso de Kubernetes **Deployments**

### Resumo de Pods

O servidor web Nginx com os arquivos estatic está rodando em dois diferentes pods. agora tenho duas perguntas:

- Como que podemos expor isso externamente para ficar acessivel via URL, e 
- Como que balanceamos a carga entre eles?

![Balanceando carga entre pods](https://cdn-media-1.freecodecamp.org/images/I4Xjozhym548e8iBKMcPJ5DnUXZojwrnmQpT)

Img. 15.Balanceando carga entre pods

Kubernetes nos prove o recurso **Services**. Vamos ir logo nele, na proxima sessão.
