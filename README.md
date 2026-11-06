# Nginx — Conexão Tech

Projeto de estudo do curso Conexão Tech para praticar Docker Compose, proxy reverso, HTTPS e balanceamento de carga com Nginx.

## Arquitetura

O ambiente tem quatro contêineres na rede bridge `frontend`:

| Serviço | Porta no contêiner | Função |
| --- | --- | --- |
| `nginx` | 80 e 443 | Recebe as requisições, redireciona HTTP para HTTPS e encaminha para as aplicações |
| `node` | 3000 | Aplicação Node.js que responde com “Hello from Node 1” e o IP do cliente |
| `node2` | 3000 | Segunda aplicação Node.js, que responde com “Hello from Node 2” e o IP do cliente |
| `python` | 5000 | Aplicação Flask que mostra o IP recebido e o host da requisição |

Somente as portas 80 e 443 do Nginx são publicadas na máquina. As aplicações são acessadas pela rede interna do Docker, usando os nomes dos serviços.

```text
Navegador
    |
    | domínio resolvido pelo /etc/hosts
    v
Máquina do laboratório — portas 80/443
    |
    v
Nginx
    ├── vini-node.conexaotech.com ──────────> node:3000
    ├── vini-python.conexaotech.com ────────> python:5000
    └── vini-balanceador.conexaotech.com ───> pool_apps
                                               ├── node:3000
                                               └── node2:3000
```

O Nginx escolhe o destino pelo domínio configurado em `server_name`. Ele recebe a conexão HTTPS usando o certificado e a chave montados pelo Compose; a comunicação com as aplicações internas usa HTTP.

## Domínios no /etc/hosts

No laboratório, adicionamos estas entradas ao `/etc/hosts` da máquina usada para acessar as aplicações:

```text
10.62.195.155 vini-node.conexaotech.com
10.62.195.155 vini-python.conexaotech.com
10.62.195.155 vini-balanceador.conexaotech.com
```

Os três nomes apontam para o mesmo IP: a máquina que publica as portas do Nginx. Não são os IPs internos dos contêineres. O que muda o destino da requisição é o domínio utilizado.

Essa alteração resolve os nomes localmente; ela não registra os domínios em um DNS público. Para reproduzir em outra máquina, substitua `10.62.195.155` pelo IP da máquina que executa o Docker e configure o arquivo hosts de cada cliente. Se o acesso for na própria máquina do Docker, também é possível usar `127.0.0.1`.

## Balanceamento de carga

A configuração em `arquivos/nginx_files/nginx/conf.d/vini-balanceador.conexaotech.com.conf` reúne as duas aplicações Node no grupo `pool_apps`:

```nginx
upstream pool_apps {
    server node:3000;
    server node2:3000;
}
```

O domínio do balanceador encaminha as requisições para esse grupo com `proxy_pass http://pool_apps/`. Como não definimos outro método, o Nginx usa round-robin: distribui as requisições entre os dois servidores. As respostas “Hello from Node 1” e “Hello from Node 2” ajudam a identificar qual aplicação atendeu cada acesso.

O domínio `vini-node.conexaotech.com` acessa somente o serviço `node`; ele não passa pelo balanceamento.

Nas rotas Node, o Nginx envia o endereço do cliente no cabeçalho `X-Real-IP`, que as aplicações leem para exibir o IP. Na rota Python, a configuração atual preserva o cabeçalho `Host`, mas não encaminha `X-Forwarded-For`; por isso, o IP mostrado pelo Flask pode ser o do proxy.

## Executando

Com Docker e Docker Compose disponíveis, execute na raiz do repositório:

```bash
docker compose up -d --build
docker compose ps
```

Com o arquivo hosts configurado, acesse:

- https://vini-node.conexaotech.com
- https://vini-python.conexaotech.com
- https://vini-balanceador.conexaotech.com

As requisições HTTP são redirecionadas para HTTPS com status 301. O navegador pode apresentar um aviso caso o certificado do laboratório não seja confiável ou não cubra o domínio acessado.

Para observar a distribuição das respostas pelo terminal:

```bash
for i in 1 2 3 4 5 6; do
    curl -ksS https://vini-balanceador.conexaotech.com
    echo
done
```

A opção `-k` ignora a validação do certificado nesse teste local.

Para validar a configuração, acompanhar os logs e encerrar o ambiente:

```bash
docker compose exec nginx nginx -t
docker compose logs -f nginx
docker compose down
```

## Organização dos arquivos

```text
arquivos/
├── nginx_files/
│   ├── certs/              # Certificado e chave do laboratório
│   └── nginx/
│       ├── nginx.conf      # Configuração principal
│       └── conf.d/         # Proxy por domínio e balanceamento
├── node_app/               # Node 1 e seu Dockerfile
├── node_app2/              # Node 2 e seu Dockerfile
└── python_app/             # Flask e seu Dockerfile
docker-compose.yml          # Serviços, volumes, portas e rede
```
