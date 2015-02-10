# RPCGEN
Es un pre-compilador generador de interfaces en C desarrollado por Sun. A partir de una especificación se crea código en C que permite inicializar un cliente y servidor.
>En 1984 Birrell y Nelson introducen una forma novedosa para administrar la comunicación entre un cliente y servidor, con una idea realmente simple. Ellos proponen permitir a los programas llamar procedimientos localizados en otras computadoras, cuando un proceso en una máquina (A) llama a un proceso en un máquena (B) el proceso en (A) es suspendido, la ejecución completa del procedimiento toma lugar en (B). La información es enviada en parametros y regresa a través del resultado del proceso, ningún tipo de envío de mensajes es visible por el programador. A ésta metodología se le conoce como Remote Procedure Call, RPC.

### Herramientas Necesarias
* Sun RPCGEN.
* GCC Compiler.
* Ubuntu Linux.

### Instrucciones de instalación:
En la terminal de Ubuntu ejecutar los siguientes comandos:
```sh
$ sudo apt-get install build-essential
$ sudo apt-get install rpcgen
$ sudo apt-get install rpcbind
```
# Actividad 1
* Crear una aplicación distribuída usando **RPCGEN**.
* Reconocer el proceso de distribución de aplicaciones en general para abordar situaciones más complejas.

### 1. Crear la interfaz que define al RPC
Dentro de la carpeta **distribuida** en el repositorio https://github.com/Innova4D/RPCGen se encuentra el archivo **suma.x**
Revisar el archivo **suma.x**, en este archivo se especifican las variables y el comportamiento del cliente-servidor.  
#### 2. Generar el código del cliente y servidor
Estos comandos generan el esqueleto de la aplicación distribuída, en donde podemos observar que se han creado los archivos para inicializar un cliente y un servidor.  
```sh
$ rpcgen -a -C suma.x
```
La bandera -C (mayúscula C) le dice a **rpcgen** que genere código C bajo la norma ANSI. Esto es lo habitual en algunas versiones de **rpcgen**. Si nos fijamos en los archivos que se han generado, podemos a ver:
#####suma.h
Este es el archivo de cabecera que incluiremos en tanto nuestro cliente y código de servidor. Define la estructura que definimos **(intpair)** y **typedefs** a un tipo del mismo nombre. También define **SUMA_PROG**, símbolos (0x23451111, nuestro número de programa) y **SUMA_VERS** (1, nuestro número de versión).
Se define la interfaz del cliente **(suma_1)** y la interfaz para la función de servidor que vamos a tener que escribir **(suma_1_svc)**.
#####suma_svc.c
Este es el programa servidor. Si nos fijamos en el código, verás que implementa el procedimiento principal, que registra al servicio.
La función llamada **suma_prog_1** contiene un statement interruptor para todos los procedimientos remotos soportados por este programa y esta versión. Además del procedimiento nulo, la única entrada en la sentencia switch es **SUMA**, esto establece un puntero a función (local) a la función de servidor, **suma_1_svc**. Más tarde, en el procedimiento, la función se invoca con el parámetro unmarshaled y la información del solicitante.

#####suma_clnt.c
Esta es la función stub del cliente que implementa la función **suma_1**, crea el parámetro, llama al procedimiento remoto, y devuelve el resultado.
#####suma_xdr.c
El archivo **_xdr.c** no siempre se genera depende de los parámetros utilizados para procedimientos remotos. Este archivo contiene código para reunir parámetros para la estructura **intpair**. Utiliza **XDR (External Data Representation)** para convertir los dos enteros a un formato estándar.

#### 3. Primera prueba
Modificamos el archivo **suma_server.c**, localizamos la línea en donde se menciona *"Insert server code here"* y agregamos lo siguiente:
```c
/*
 * insert server code here
 */
printf("Server response to client...\n");
```
Éste mensaje nos permitirá saber cuando el servidor reciba un mensaje desde algún cliente.
Antes de compilar debemos hacer algunas modificaciones al archivo **Makefile.suma** principalmente por que deseamos que el proceso del servidor se ejecute en segundo plano y además asegurar que el código generado corresponda a **ANSI-C**.
De este modo, le indicamos a **RPCGEN** que ejecute en segundo plano:
```sh
CFLAGS += -g
```
Cambiarlo a:
```sh
CFLAGS += -g -DRPC_SVC_FG
```
Nos aseguramos que **RPCGEN** genere código ANSI-C:
```sh
RPCGENFLAGS =
```
Cambiarlo a:
```sh
RPCGENFLAGS = -C
```

A continuación ejecutamos **Makefile.suma** esto nos compilará todo el código de nuestro proyecto usando el compilador de **C**.
```sh
$ mv Makefile.suma makefile
$ make
```
Para nuestro caso inicializaremos un servidor en nuestra misma computadora utilizando *portmap*, en un puerto, ejecutamos suma_server con permisos de administrador:
```sh
$ sudo ./suma_server
```
Abrimos una nueva terminal en una nueva pestaña ó ventana y ahora ejecutaremos al cliente y especificamos el lugar en donde se encuentra nuestro servidor:
```sh
$ ./suma_client localhost
```
Si revisamos la terminal en donde se encuentra ejecutándose el proceso del servidor podremos ver el mensaje **"Server response to client..."**, con esto podemos confirmar que la comunicación entre el cliente y el servidor es satisfactoria y estamos llamando remotamente a nuestro proceso.

Si quisieramos llamar el proceso desde otra computadora tendríamos que especificar la ubicación del cliente en lugar de **"localhost"**.

#### 4. Pongamos a trabajar al servidor
Ahora que verificamos que el servidor y el cliente se comunican es momento de poner a trabajar al servidor. Editamos el archivo **suma_client.c**. Aquí debemos observar que la función **suma_prog_1** define la variable **suma_1_arg** este parámetro se envía al proceso remoto unas líneas después mediante:
```sh
result_1 = suma_1(&suma_1_arg, clnt);
```
Agregamos:
```sh
    suma_1_arg.a = 123;
    suma_1_arg.b = 22;
```
Ejemplo:
```sh
	if (clnt == NULL) {
		clnt_pcreateerror (host);
		exit (1);
	}
#endif	/* DEBUG */
    suma_1_arg.a = 123;
    suma_1_arg.b = 22;  
	result_1 = suma_1(&suma_1_arg, clnt);
	if (result_1 == (int *) NULL) {
		clnt_perror (clnt, "call failed");
	}
```
En terminal podemos ejecutar **make** para compilar todo y asegurarnos que no hay errores:
```sh
$ make
```
A continuación en el archivo **suma_server.c** vamos a imprimir los parámetros recibidos a través de **argp**:
```sh
/*
 * insert server code here
 */
printf("Server is working \n");
printf("parameters: %d, %d\n", argp->a, argp->b);
return &result;
```
Compilar con:
```sh
$ make
```
Ejecutar el servidor y después el cliente (como en el paso 3). En la pantalla debería aparecer ahora:
```sh
Server response to client...
parameters: 123, 22
```
Esto confirma que los parámetros han sido enviados correctamente desde el cliente. Ahora en **suma_server.c** vamos a computar el resultado y enviarlo de vuelta.
```sh
printf("Server response to client...\n");
printf("parameters: %d, %d\n", argp->a, argp->b);
result = argp->a + argp->b;
printf("returning: %d\n", result);
return &result;
```
Aquí es importante destacar que la variable **result** es estática debido a que las variables locales viven en el stack, esto es un problema por que el espacio en memoria puede ser solicitado para ser utilizado por el servidor y causar errores.
Ahora, en el cliente, **suma_client.c** agregaremos el código para imprimir el resultado:
```sh
if (result_1 == (int *) NULL) {
	clnt_perror (clnt, "call failed");
} else {
	printf("result = %d\n", *result_1);
}
```
Compilar con **make**, ejecutar el **servidor** y **cliente** nuevamente.
Ahora deberíamos observar los resultados en el **servidor** y la **respuesta** en el **cliente**.

Ahora tenemos un servidor funcional.
#### 5. Haciendo un mejor cliente
Para tener un mejor cliente debemos hacer que nuestro programa sume dos numeros directamente desde la consola, actualmente nuestros números se ingresan desde el código fuente, para lograr esto debemos hacer algunos cambios en **suma_client.c**:  
Cambiamos la faunción **suma_prog_1** para que acepte dos parámetros:
```c
void
suma_prog_1(char *host, int a, int b)
```
Asignamos estos parámetros a las variables del proceso remoto en:
```c
suma_1_arg.a = a;
suma_1_arg.b = b;
result_1 = suma_1(&suma_1_arg, clnt);
```
Antes de compilar no olvidemos añadir la librería ** stdio.h** al inicio del archivo.  
```c
#include <stdio.h>
```
Debemos hacer los cambios correspondientes en la función **main**, debería verse así:
```c
int
main(int argc, char *argv[]) {
        char *host;
        int a, b;
        if (argc != 4) {
            printf ("usage: %s server_host num1 num2\n", argv[0]);
            exit(1);
        }
        host = argv[1];
        if ((a = atoi(argv[2])) == 0 && *argv[2] != '0') {
            fprintf(stderr, "invalid value: %s\n", argv[2]);
            exit(1);
        }
        if ((b = atoi(argv[3])) == 0 && *argv[3] != '0') {
            fprintf(stderr, "invalid value: %s\n", argv[3]);
            exit(1);
        }
        suma_prog_1(host, a, b);
}
```
Compilar con **make**, ejecutar el **servidor** y **cliente** nuevamente. Ahora el cliente si lo ejecutamos de este modo:
```sh
./suma_client localhost 5 7
```
Debería mostrar:
```sh
result = 12
```
¡El programa funciona completamente, felicidades!
#### 6. Limpieza
Ésta es la parte mas importante de la actividad, pues limpiar el código nos permitirá comprender de mejor manera la esctructura del **RPC**.

* Identifica los archivos de cliente y servidor generados por RPCGen, comenta en el código tus observaciones. ¿Los nombres de variables son comprensibles?
* Identifica la función main. ¿Qué hace? ¿Podrías hacerlo de otra manera? Comenta el código al respecto.
* ¿Cómo maneja los errores RPCGen? ¿Qué pasa si hay un error y se siguen haciendo llamadas a través de RPC? Comenta en el código en los archivos correspondientes.
* Elimina todos los comentarios generados por RPCGen que indiquen que los archivos son un template. Agrega al inicio de cada archivo .c tus propios comentarios al respecto.
* Las líneas con **#ifdef DEBUG** deberían quitarse. ¿Qué puede hacerse para que el código se más legible?

Anota tus comentarios al respecto en el **código** y en tu **reporte**.

#### Ejemplos:
```sh
/* RPC example: suma two numbers */

#include "suma.h"
CLIENT *rpc_setup(char *host);
void suma(CLIENT *clnt, int a, int b);

int
main(int argc, char *argv[])
{
	CLIENT *clnt;  /* client handle to server */
	char *host;    /* host */
	int a, b;

	if (argc != 4) {
		printf("usage: %s server_host num1 num2\n", argv[0]);
		exit(1);
	}
	host = argv[1];
	if ((a = atoi(argv[2])) == 0 && *argv[2] != '0') {
		fprintf(stderr, "invalid value: %s\n", argv[2]);
		exit(1);
	}
	if ((b = atoi(argv[3])) == 0 && *argv[3] != '0') {
		fprintf(stderr, "invalid value: %s\n", argv[3]);
		exit(1);
	}
	if ((clnt = rpc_setup(host)) == 0)
		exit(1);	/* cannot connect */
	suma(clnt, a, b);
	clnt_destroy(clnt);
	exit(0);
}

CLIENT *
rpc_setup(char *host)
{
	CLIENT *clnt = clnt_create(host, suma_PROG, suma_VERS, "udp");
	if (clnt == NULL) {
		clnt_pcreateerror(host);
		return 0;
	}
	return clnt;
}

void
suma(CLIENT *clnt, int a, int b)
{
	int  *result;
	intpair v;	/* parameter for suma */

	v.a = a;
	v.b = b;
	result = suma_1(&v, clnt);
	if (result == 0) {
		clnt_perror(clnt, "call failed");
	} else {
		printf("%d\n", *result);
	}
}
```
```sh
/*
 * RPC server code for the remote suma function
 */

#include "suma.h"

int *
suma_1_svc(intpair *argp, struct svc_req *rqstp)
{
	static int  result;

	result = argp->a + argp->b;
	printf("suma(%d, %d) = %d\n", argp->a, argp->b, result);
	return &result;
}
```
### One last thing...
Revisemos el makefile, desafortunadamente el generado por RPCGen no es fácil de comprender, revisemos uno creado por nosotros mismos. Anota en el reporte tus comentarios al respecto
```sh
CC = gcc
CFLAGS = -g -DRPC_SVC_FG
RPCGEN_FLAG = -C

all: suma_client suma_server

# the executables: suma_client and suma_server

suma_client: suma_client.o suma_clnt.o suma_xdr.o
	$(CC) -o suma_client suma_client.o suma_clnt.o suma_xdr.o -lnsl

suma_server: suma_server.o suma_svc.o  suma_xdr.o
	$(CC) -o suma_server suma_server.o suma_svc.o suma_xdr.o -lnsl

# object files for the executables

suma_server.o: suma_server.c suma.h
	$(CC) $(CFLAGS) -c suma_server.c

suma_client.o: suma_client.c suma.h
	$(CC) $(CFLAGS) -c suma_client.c

# compile files generated by rpcgen

suma_svc.o: suma_svc.c suma.h
	$(CC) $(CFLAGS) -c suma_svc.c

suma_clnt.o: suma_clnt.c suma.h
	$(CC) $(CFLAGS) -c suma_clnt.c

suma_xdr.o: suma_xdr.c suma.h
	$(CC) $(CFLAGS) -c suma_xdr.c

# suma.x produces suma.h, suma_clnt.c, suma_svc.c, and suma_xdr.c
# make sure we regenerate them if our interface (suma.x) changes

suma_clnt.c suma_svc.c suma_xdr.c suma.h:	suma.x
	rpcgen $(RPCGEN_FLAG) suma.x

clean:
	rm -f suma_client suma_client.o suma_server suma_server.o suma_clnt.* suma_svc.* suma.h
```

# Actividad 2
Ahora que ya sabemos como trabajar con RPCGen vamos a generar una nueva aplicación, los objetivos son los siguientes:  
* Generar un nueva especificación RPCGen (Como add.x) en donde se definan dos funciones:
  * La función "agregar" deberá:
    * Recibir un nombre y obtener la fecha con la librería [Date](http://goo.gl/tSrZ46) de C.
    * Guardar en un archivo .txt la fecha y el nombre.
    * Retornar un mensaje si los datos se han escrito satisfactoriamente.
  * La función "buscar" deberá:
    * Recibir un nombre, mediante una búsqueda obtener la fecha relacionada al nombre.
    * Retornar el la fecha relacionada con el nombre indicado, caso contrario enviar un mensaje de inexistencia.

Para la parte del cliente, se debe modificar la estructura del main y adecuar el código para ejecutar las funciones y enviar los parámetros.

Aquí un ejemplo de la definición de dos funciones con RPCGen:

```C
program ADD_PROG {
  version ADD_VERS {
    string ADD(string) = 1;
    string SEARCH(string) = 2;
   } = 1;
} = 0x23451111;
```
