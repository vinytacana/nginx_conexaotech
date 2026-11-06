#Define a imagem base do container, nesse caso será a versão 24 do Node.js
FROM node:24

#Configura o diretório padrão dentro do container. Qualquer comando seguinte será executado dentro dessa pasta.
WORKDIR /root

#Pega o arquivo server.js que está na máquina local (na pasta ./arquivos/) e o copia para o diretório atual do container.
COPY ./arquivos/server.js .

#Informa que o container vai escutar requisições na porta 3000 quando estiver rodando.
EXPOSE 3000

#É o comando principal que será disparado assim que o container for iniciado. Ele executa o interpretador do Node para rodar o arquivo.
ENTRYPOINT ["node", "server.js"]
