# Despliegue de una aplicación en "Cluster" con NodeJS y Express

## Contenidos
- [Preparación de la máquina](#preparación-de-la-máquina)
- [Aplicación sin clústeres](#aplicación-sin-clústeres)
- [Aplicación con clústeres](#aplicación-con-clústeres)
- [Métricas de rendimiento](#métricas-de-rendimiento)
    - [Métricas de la aplicación sin clústeres](#métricas-de-la-aplicación-sin-clústeres)
    - [Métricas de la aplicación con clústeres](#métricas-de-la-aplicación-con-clústeres)
- [PM2](#pm2)
    - [Tarea](#tarea)
- [Cuestión final](#cuestión-final)

## Preparación de la máquina

Se prepara una provisión simple en Debian 12 a través de *Vagrantfile*:
```Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "practica" do |p|
      p.vm.box = "debian/bookworm64"
      p.vm.hostname = "practica"
      p.vm.network "forwarded_port", guest: 8080, host: 8080
      p.vm.network "private_network", ip: "192.168.12.12"
  end # practica
end # cofig
```

Se levanta la máquina:  
`vagrant up`

Se conecta al servidor mediante SSH:  
`vagrant ssh`

Actualizar los repositorios:  
`sudo apt update`

Instalar node.js:  
`sudo apt install nodejs npm -y`

## Aplicación sin clústeres

En el directorio personal *home/vagrant*:  
`mkdir noClusterApp`

Dentro del directorio creado:  
```bash
npm init

npm install express
```

<img src="./assets/1.png">

Se crea la aplicación:  
`nano noCluster.js`

Contenido:
```bash
const express = require("express");
const app = express();
const port = 3000;
const limit = 5000000000;

app.get("/", (req, res) => {
        res.send("Hello World!");
});

app.get("/api/:n", function (req, res) {
        let n = parseInt(req.params.n);
        let count = 0;

        if (n > limit) n = limit;

        for (let i = 0; i <= n; i++) {
                count += i;
        }

        res.send(`Final count is ${count}`);
});

app.listen(port, () => {
        console.log(`App listening on port ${port}`);
});
```

Se inicia la aplicación:  
`node noCluster.js`

<img src="./assets/2.png">

Accediendo a *http://192.168.12.12:3000*:
<img src="./assets/3.png">

Accediendo a *http://192.168.12.12:3000/api/50*:
<img src="./assets/4.png">

Accediendo a *http://192.168.12.12:3000/api/5000000000*:
<img src="./assets/5.png">

Comparación tiempo que tardan las solicitudes en procesarse:
<img src="./assets/6.png">
<img src="./assets/7.png">
<img src="./assets/8.png">

## Aplicación con clústeres

En el directorio personal *home/vagrant*:  
`mkdir clusterApp`

Dentro del directorio creado:  
```bash
npm init

npm install express
```

Se crea la aplicación:  
`nano cluster.js`

Contenido:
```bash
const express = require("express");
const port = 3000;
const limit = 5000000000;
const cluster = require("cluster");
const totalCPUs = require("os").cpus().length;

if (cluster.isMaster) {
    console.log(`Number of CPUs is ${totalCPUs}`);
    console.log(`Master ${process.pid} is running`);
    // Fork workers.
    for (let i = 0; i < totalCPUs; i++) {
        cluster.fork();
    }

    cluster.on("exit", (worker, code, signal) => {
        console.log(`worker ${worker.process.pid} died`);
        console.log("Let's fork another worker!");
        cluster.fork();
    });
} else {
    const app = express();
    console.log(`Worker ${process.pid} started`);

    app.get("/", (req, res) => {
        res.send("Hello World!");
    });

    app.get("/api/:n", function (req, res) {
        let n = parseInt(req.params.n);
        let count = 0;

        if (n > limit) n = limit;

        for (let i = 0; i <= n; i++) {
            count += i;
        }

        res.send(`Final count is ${count}`);
    });

    app.listen(port, () => {
        console.log(`App listening on port ${port}`);
    });
}
```

Se inicia la aplicación:  
`node cluster.js`

<img src="./assets/9.png">

Accediendo a *http://192.168.12.12:3000/api/5000000000*:
<img src="./assets/10.png">

Accediendo a *http://192.168.12.12:3000/api/50*:
<img src="./assets/11.png">

## Métricas de rendimiento

Se instala globalmente *loadtest* en */home/vagrant*:  
`sudo npm install -g loadtest`

### Métricas de la aplicación sin clústeres

Se ejecuta la aplicación a probar:  
`node noClusterApp/noCluster.js`

En otra terminal, accediendo a la máquina de nuevo, se realiza la prueba de carga:  
`loadtest http://localhost:3000/api/500000 -n 1000 -c 100`
<img src="./assets/12.png">

`loadtest http://localhost:3000/api/500000000 -n 1000 -c 100`
<img src="./assets/13.png">

### Métricas de la aplicación con clústeres

Se ejecuta la aplicación a probar:  
`node clusterApp/cluster.js`

En otra terminal, accediendo a la máquina de nuevo, se realiza la prueba de carga:  
`loadtest http://localhost:3000/api/500000 -n 1000 -c 100`
<img src="./assets/14.png">

`loadtest http://localhost:3000/api/500000000 -n 1000 -c 100`
<img src="./assets/15.png">

## PM2

Se instala globalmente:  
`sudo npm install pm2 -g`

Se utiliza con la aplicación sin cluster:  
`pm2 start noClusterApp/noCluster.js -i 0`  
*-i* le indicará a PM2 que inicie la aplicación en *cluster_mode*
<img src="./assets/16.png">

Se detiene la aplicación:  
`pm2 stop noClusterApp/noCluster.js`
<img src="./assets/17.png">

Crear el archivo *ecosystem.config.js* para guardar la configuración:  
`pm2 ecosystem`  
Se edita:  
`nano /home/vagrant/ecosystem.config.js`  
Contenido:
```bash  
module.exports = {
    apps: [{
        name: "noClusterApp",
        script: "noClusterApp/noCluster.js",
        instances: 0,
        exec_mode: "cluster",
    },
    ],
};
```

Valores de *-i* o *instances*:
- **0** o **max**: para "repartir" la aplicación entre todas las CPU (en desuso)
- **-1**: para "repartir" la aplicación en todas las CPU - 1
- **número**: para difundir la aplicación a través de un número concreto de CPU

Ahora se puede ejecutar la aplicación con:  
`pm2 start ecosystem.config.js`
<img src="./assets/18.png">

Comandos permitidos para este modo de ejecución:
```bash
$ pm2 start nombre_aplicacion
$ pm2 restart nombre_aplicacion
$ pm2 reload nombre_aplicacion
$ pm2 stop nombre_aplicacion
$ pm2 delete nombre_aplicacion

# Cuando usemos el archivo Ecosystem:
$ pm2 [start|restart|reload|stop|delete] ecosystem.config.js
```

### Tarea

`pm2 ls`

Muestra una lista de todas las aplicaciones gestionadas por PM2.
<img src="./assets/19.png">

`pm2 logs`

Muestra los registros (logs) en tiempo real de todas las aplicaciones gestionadas por PM2.
<img src="./assets/20.png">

`pm2 monit noClusterApp/noCluster.js`

Abre una interfaz de monitoreo interactiva en la terminal que muestra el estado en tiempo real de todas las aplicaciones gestionadas por PM2. Esta interfaz incluye información sobre el uso de CPU, memoria, y otros detalles de rendimiento de cada aplicación.
<img src="./assets/21.png">

## Cuestión final

**¿Por qué en algunos casos concretos, la aplicación sin clusterizar puede tener mejores resultados?**

Por tener menor latencia, lo que facilita respuestas más rápidas y genera menos sobrecarga.

Esta menor latencia viene del hecho de que sea un solo proceso el que gestiona todas las solicitudes, de manera que se ahorran tiempo y recursos al no tener que gestionar la comunicación entre varios procesos y la distribución de la tarea entre los mismos.