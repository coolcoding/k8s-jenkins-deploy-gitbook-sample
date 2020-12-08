FROM node:10.23 AS build

WORKDIR /gitbook

COPY . ./

RUN npm install gitbook-cli -g

RUN gitbook install

RUN gitbook build

FROM nginx:alpine

COPY --from=build /gitbook/_book /var/www

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

ENTRYPOINT ["nginx","-g","daemon off;"]
