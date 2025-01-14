# Nodejs app using OAuth mtlS with Cloudentity Authorization platform

Cloudentity authorization platform completely supports the RFC8705(https://datatracker.ietf.org/doc/html/rfc8705) for OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens.
As you might already be aware Cloudentity platform is compliant to latest emerging OAuth specifications and can support in modernizing the application architecutres with latest open standards and
specfications support. We will take you down the path of understanding use cases that can be addressed using mTLS specification, code samples in various language on how to integrate and utilize the latest
specification in your new architecture patterns.

In this specific section , we will be creating a Nodejs application and calling a resource server protecting a resource with mTLS bound accessToken
This article is geared towards showing a backend application calling another service

## Pre-requisites

* Nodejs
* Npm

##

Import required packages


``` bash
var axios = require('axios');
var qs = require('qs');
var express = require('express');
var app = express();
var jwt_decode = require('jwt-decode');
var fs = require('fs');
var https = require('https');

var mustacheExpress = require('mustache-express');
var bodyParser = require('body-parser');
var path = require('path')
require('dotenv').config();
```

Set up express app to serve some views and html pages

```bash
app.set('views', `${__dirname}/views`);
app.set('view engine', 'mustache');
app.engine('mustache', mustacheExpress());
app.use (bodyParser.urlencoded( {extended : true} ) );
app.use(express.static(path.join(__dirname, "/public")));
```

Set up routes to serve traffic

Route for home page
```bash

app.get('/home', function(req, res) {
  res.render('home', {pageTitle: "Enter Your Name"} )
})

```

Route for fetching a regular access Token

```bash

app.get('/auth', function(req, res) {
  getAuth().then(value => {
   if(value !== undefined) {
     var decoded = jwt_decode(value);
    res.render('home', {accessToken: JSON.stringify(decoded, null, 4)} )
   } else {
     res.send("No token fetched!")
   }
 }, err => {
   res.send("Unable to fetch token!")
 })
 
});

const client_id = process.env.OAUTH_CLIENT_ID; // Your client id
const client_secret = process.env.OAUTH_CLIENT_SECRET; // Your secret
const token_url = process.env.OAUTH_TOKEN_URL; // Your secret
const auth_token = Buffer.from(`${client_id}:${client_secret}`, 'utf-8').toString('base64');

const httpsAgent = new https.Agent({
  cert: fs.readFileSync('full_chain.pem'),
  key: fs.readFileSync('acp_key.pem'),
});


const getAuth = async () => {
try {
 const data = qs.stringify({'grant_type':'client_credentials'});
 const response = await axios.post(token_url, data, {
   headers: { 
     'Authorization': `Basic ${auth_token}`,
     'Content-Type': 'application/x-www-form-urlencoded' 
   }
 })
 return response.data.access_token; 
}catch(error){
 console.log(error);
}
}

```

Route for fetching a certificate bound access Token

```bash

const mtls_client_id = process.env.MTLS_OAUTH_CLIENT_ID; // Your client id
const mtls_client_secret = process.env.MTLS_OAUTH_CLIENT_SECRET; // Your secret
const mtls_token_url = process.env.MTLS_OAUTH_TOKEN_URL; // Your secret
const mtls_auth_token = Buffer.from(`${mtls_client_id}:${mtls_client_secret}`, 'utf-8').toString('base64');

const getMtlsAuth = async () => {
  try{
    const data = qs.stringify({'grant_type':'client_credentials', 'client_id': mtls_client_id});

    const httpOptions = {
      url: mtls_token_url,  
      method: "POST",
      httpsAgent : httpsAgent,
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded' 
    },
    data: data
    }

    const response = await axios(httpOptions)
    return response.data.access_token;
  } catch(error){
    console.log(error);
  }
}

```

Route to call a sample resource server url

```bash

```











const resource_url = process.env.MTLS_RESOURCE_URL; // Your secret

const callResourceServerMtlsAPi = async (accessToken) => {
  try {
   const httpOptions = {
    url: resource_url,  
    method: "GET",
    httpsAgent : httpsAgent,
    headers: {
      'Content-Type': 'application/json' ,
      'Authorization': 'Bearer ' + accessToken
    }
  }
  
  const result = await axios(httpOptions)
    return result.data;  
  }catch(error){
    console.log(error);
  }

}



app.get('/mtlsauth', function(req, res) {
  getMtlsAuth().then(value => {
   if(value !== undefined) {
     var decoded = jwt_decode(value);
     res.render('home', {certificate_bound_access_token: JSON.stringify(decoded, null, 4)} )
   } else {
     res.send("No token fetched!")
   }
 }, err => {
   res.send("Unable to fetch token!")
 })
});



app.get('/mtls-resource', function(req, res) {
  getMtlsAuth().then(value => {
    if(value !== undefined) {
        var decoded = jwt_decode(value);
        callResourceServerMtlsAPi(value).then(value => {
        if(value !== undefined) {
          res.render('home', {mtls_resource: JSON.stringify(value, null, 4)} )
        } else {
          res.send("No response fetched!")
        }
      }, err => {
        res.send("Unable to fetch token!")
      })
  }
  });
});

app.get('/mtls-resource-roguecaller', function(req, res) {
  getMtlsAuth().then(value => {
    if(value !== undefined) {
        var decoded = jwt_decode(value);
        callResourceServerMtlsAPiAsRogueCaller(value).then(value => {
        if(value !== undefined) {
          res.render('home', {mtls_resource_rogue_caller: value} )
        } else {
          res.send("No response fetched!")
        }
      }, err => {
        res.send("Unable to fetch token!")
      })
  }
  });
});

const rogueHttpsAgent = new https.Agent({
  cert: fs.readFileSync('api-server-cert.pem'),
  key: fs.readFileSync('api-server-key.pem'),
});


const callResourceServerMtlsAPiAsRogueCaller = async (accessToken) => {
  try {
    const httpOptions = {
    url: resource_url,  
    method: "GET",
    httpsAgent : rogueHttpsAgent,
    headers: {
      'Content-Type': 'application/json' ,
      'Authorization': 'Bearer ' + accessToken
    }
  }
 
  const result = await axios(httpOptions)
  return result;
   
  } catch(error)
  {
     console.log(error);
   return error;

  }

}


app.get('/health', function(req, res) {
    res.send('Service is alive and healthy')
});

app.listen(5002);
console.log("Server listening at http://localhost:5002/");

```

##


