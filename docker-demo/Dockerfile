FROM node:9

COPY ./expressjs/ /app/
WORKDIR app/expressjs

RUN npm i

EXPOSE 3000

ENTRYPOINT [ "npm" ]
CMD [ "start" ]