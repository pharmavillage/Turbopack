Monorepo Development Environment Setup with Turbopack, Docker, NX Workspaces, and Module Federation

This README provides a step-by-step guide for setting up a monorepo development environment using Turbopack, Docker Compose, NX Workspaces, and Module Federation. The environment will integrate legacy code, Vue, and React applications, utilizing Go as a wrapper for legacy code and using GraphQL for data manipulation.

Prerequisites

Ensure you have the following installed on your development machine:

Docker
Docker Compose
Node.js (preferably via nvm)
Go
NX CLI (npm install -g nx)
Project Structure

The monorepo will have the following structure:

bash
Copy code
/monorepo
  ├── /apps
  │   ├── /react-app
  │   ├── /vue-app
  ├── /legacy
  ├── /services
  │   ├── /go-wrapper
  ├── /libs
  ├── /docker
  │   ├── /react-app
  │   ├── /vue-app
  │   ├── /go-wrapper
  ├── /graphql
  │   ├── schema.graphql
  ├── docker-compose.yml
  ├── nx.json
  ├── workspace.json
  ├── package.json
Setting Up Docker Compose

Create a docker-compose.yml file in the root of your project:

yaml
Copy code
version: '3.8'
services:
  react-app:
    build:
      context: ./apps/react-app
      dockerfile: ../../docker/react-app/Dockerfile
    volumes:
      - ./apps/react-app:/app
    ports:
      - "3000:3000"
  
  vue-app:
    build:
      context: ./apps/vue-app
      dockerfile: ../../docker/vue-app/Dockerfile
    volumes:
      - ./apps/vue-app:/app
    ports:
      - "8080:8080"
  
  go-wrapper:
    build:
      context: ./services/go-wrapper
      dockerfile: ../../docker/go-wrapper/Dockerfile
    volumes:
      - ./services/go-wrapper:/app
    ports:
      - "4000:4000"
  
  graphql:
    image: node:14
    working_dir: /app
    volumes:
      - ./graphql:/app
    command: npm start
    ports:
      - "5000:5000"
Dockerfile for React App
Create Dockerfile for React app in docker/react-app:

dockerfile
Copy code
# docker/react-app/Dockerfile
FROM node:14

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "start"]
Dockerfile for Vue App
Create Dockerfile for Vue app in docker/vue-app:

dockerfile
Copy code
# docker/vue-app/Dockerfile
FROM node:14

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 8080
CMD ["npm", "run", "serve"]
Dockerfile for Go Wrapper
Create Dockerfile for Go wrapper in docker/go-wrapper:

dockerfile
Copy code
# docker/go-wrapper/Dockerfile
FROM golang:1.16

WORKDIR /app

COPY go.mod .
COPY go.sum .
RUN go mod download

COPY . .

RUN go build -o main .

EXPOSE 4000
CMD ["./main"]
Setting Up NX Workspaces

Create the NX workspace:

sh
Copy code
npx create-nx-workspace@latest monorepo
Move your existing apps and libs into the apps and libs directories respectively.

Update the workspace.json and nx.json files to include your projects.

Create project.json files for each application to define how they should be built, served, and tested.

Example project.json for React App
json
Copy code
{
  "name": "react-app",
  "root": "apps/react-app",
  "sourceRoot": "apps/react-app/src",
  "projectType": "application",
  "targets": {
    "build": {
      "executor": "@nrwl/web:build",
      "options": {
        "outputPath": "dist/apps/react-app",
        "index": "apps/react-app/src/index.html",
        "main": "apps/react-app/src/main.tsx",
        "polyfills": "apps/react-app/src/polyfills.ts",
        "tsConfig": "apps/react-app/tsconfig.app.json",
        "assets": ["apps/react-app/src/favicon.ico", "apps/react-app/src/assets"]
      },
      "configurations": {
        "production": {
          "fileReplacements": [
            {
              "replace": "apps/react-app/src/environments/environment.ts",
              "with": "apps/react-app/src/environments/environment.prod.ts"
            }
          ],
          "optimization": true,
          "outputHashing": "all",
          "sourceMap": false,
          "extractCss": true,
          "namedChunks": false,
          "aot": true,
          "extractLicenses": true,
          "vendorChunk": false,
          "buildOptimizer": true,
          "budgets": [
            {
              "type": "initial",
              "maximumWarning": "2mb",
              "maximumError": "5mb"
            }
          ]
        }
      }
    },
    "serve": {
      "executor": "@nrwl/web:dev-server",
      "options": {
        "buildTarget": "react-app:build"
      }
    }
  }
}
Example project.json for Vue App
json
Copy code
{
  "name": "vue-app",
  "root": "apps/vue-app",
  "sourceRoot": "apps/vue-app/src",
  "projectType": "application",
  "targets": {
    "build": {
      "executor": "@nrwl/web:build",
      "options": {
        "outputPath": "dist/apps/vue-app",
        "index": "apps/vue-app/public/index.html",
        "main": "apps/vue-app/src/main.ts",
        "polyfills": "apps/vue-app/src/polyfills.ts",
        "tsConfig": "apps/vue-app/tsconfig.app.json",
        "assets": ["apps/vue-app/src/favicon.ico", "apps/vue-app/src/assets"]
      },
      "configurations": {
        "production": {
          "fileReplacements": [
            {
              "replace": "apps/vue-app/src/environments/environment.ts",
              "with": "apps/vue-app/src/environments/environment.prod.ts"
            }
          ],
          "optimization": true,
          "outputHashing": "all",
          "sourceMap": false,
          "extractCss": true,
          "namedChunks": false,
          "aot": true,
          "extractLicenses": true,
          "vendorChunk": false,
          "buildOptimizer": true,
          "budgets": [
            {
              "type": "initial",
              "maximumWarning": "2mb",
              "maximumError": "5mb"
            }
          ]
        }
      }
    },
    "serve": {
      "executor": "@nrwl/web:dev-server",
      "options": {
        "buildTarget": "vue-app:build"
      }
    }
  }
}
Integrating Go Wrapper and GraphQL

Create a Go wrapper service to interact with the legacy code. This service will expose a GraphQL API for data manipulation.

Install dependencies:

sh
Copy code
go get github.com/graphql-go/graphql
go get github.com/graphql-go/handler
Implement the Go wrapper in services/go-wrapper/main.go:

go
Copy code
package main

import (
    "net/http"
    "github.com/graphql-go/graphql"
    "github.com/graphql-go/handler"
)

var schema, _ = graphql.NewSchema(
    graphql.SchemaConfig{
        Query:    queryType,
        Mutation: mutationType,
    },
)

var queryType = graphql.NewObject(
    graphql.ObjectConfig{
        Name: "Query",
        Fields: graphql.Fields{
            "data": &graphql.Field{
                Type: graphql.String,
                Resolve: func(p graphql.ResolveParams) (interface{}, error) {
                    return "Hello, world!", nil
                },
            },
        },
    },
)

var mutationType = graphql.NewObject(
    graphql.ObjectConfig{
        Name: "Mutation",
        Fields: graphql.Fields{
            "updateData": &graphql.Field{
                Type: graphql.String,
                Args: graphql.FieldConfigArgument{
                    "input": &graphql.ArgumentConfig{
                        Type: graphql.String,
                    },
                },
                Resolve: func(p graphql.ResolveParams) (interface{}, error) {
                    input := p.Args["input"].(string)
                    return input, nil
                },
            },
        },
    },
)

func main() {
    http.Handle("/graphql", handler.New(&handler.Config{
        Schema: &schema,
        Pretty: true,
    }))
    http.ListenAndServe(":4000", nil)
}
Create schema.graphql in graphql directory:

graphql
Copy code
type Query {
  data: String
}

type Mutation {
  updateData(input: String): String
}
Create index.js in graphql directory to set up the GraphQL server:

javascript
Copy code
const { ApolloServer, gql } = require('apollo-server');
const { createProxyMiddleware } = require('http-proxy-middleware');

const typeDefs = gql`
  type Query {
    data: String
  }

  type Mutation {
    updateData(input: String): String
  }
`;

const resolvers = {
  Query: {
    data: () => 'Hello, world!',
  },
  Mutation: {
    updateData: (_, { input }) => input,
  },
};

const server = new ApolloServer({ typeDefs, resolvers });

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
Using Module Federation

Set up module federation in webpack.config.js for React and Vue apps.
webpack.config.js for React App
javascript
Copy code
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");

module.exports = {
  // ...other configurations
  plugins: [
    new ModuleFederationPlugin({
      name: "reactApp",
      filename: "remoteEntry.js",
      exposes: {
        "./Component": "./src/components/Component",
      },
      shared: require("./package.json").dependencies,
    }),
  ],
};
webpack.config.js for Vue App
javascript
Copy code
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");

module.exports = {
  // ...other configurations
  plugins: [
    new ModuleFederationPlugin({
      name: "vueApp",
      filename: "remoteEntry.js",
      exposes: {
        "./Component": "./src/components/Component",
      },
      shared: require("./package.json").dependencies,
    }),
  ],
};
Running the Development Environment

Build and start the services using Docker Compose:

sh
Copy code
docker-compose up --build
Access the applications:

React App: http://localhost:3000
Vue App: http://localhost:8080
Go Wrapper: http://localhost:4000
GraphQL Server: http://localhost:5000
Conclusion

This setup provides a structured and scalable development environment for a monorepo that integrates React, Vue, and legacy code, using Docker, NX Workspaces, Go, and GraphQL. With this setup, you can easily manage and develop multiple applications within a single repository.




