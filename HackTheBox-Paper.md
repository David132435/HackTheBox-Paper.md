![image](https://user-images.githubusercontent.com/106914229/172148676-99c3eaa4-7046-4877-a46b-552ab2498adc.png)
Ayer conseguí resolver mi primera máquina de la plataforma HackTheBox y me he propuesto hacer una guía con todos los pasos que seguí para poder resolverla. Paper es un máquina de nivel fácil con Sistema Operativo Linux basada en la serie The Office.

Fase de reconocimiento
Lo primero que haremos será un escaneo en busca de puertos abiertos por TCP. Para ello, he utilizado el siguiente comando : nmap -sS -p- --min-rate 5000 -n -Pn 10.10.11.143

![image](https://user-images.githubusercontent.com/106914229/172150841-33369831-19a5-46f3-adc5-b4a44438067b.png)

Como vemos en la imagen, nos encontramos con tres puertos abiertos, el número 22 que corresponde a SSH, el número 80 que corresponde a HTTP y el número 443 que corresponde a HTTPS. Lo siguiente que haremos será utlizar los parámetros -sC y -sV dee nmap para detectar la versión y servicio de esos puertos con el siguiente comando: map -sCV -p22,80,443 --min-rate 5000 -n -Pn 10.10.11.143

![image](https://user-images.githubusercontent.com/106914229/172151429-3730a20d-8202-4e1d-882c-57ed53b009e0.png)

LLegado a este punto y sabiendo que hay un servicio tanto HTTP como HTTPS podemos ir al navegador para ver el contenido de la web.

![image](https://user-images.githubusercontent.com/106914229/172151782-9859f174-61ae-4630-bb1e-7d6486b20a70.png)
![image](https://user-images.githubusercontent.com/106914229/172151872-66c47394-cee8-4176-b5aa-7c4c7cd6e6ba.png)

Tanto por HTTP como por HTTPS nos encontramos con lo mismo. La página parece estar en mantenimiento y no hay mucho que podamos hacer. Este es un punto en el que es fácil quedarse atascado, sin embargo hay algo que podemso hacer. Utlizando el comando curl con la opción -I hará que solo se nos muestren las cabeceras, lo cual nos dará información que nos interesa:
![image](https://user-images.githubusercontent.com/106914229/172153623-e256f9e8-9e19-40be-8e39-3e1f413b0bc4.png)
Si miramos la linea que dice "X-Backend-Server" podemos ver un nombre de dominio. Quizás en este nombre de dominio encontremos un hilo del que tirar asi que vamos a asociar ese nombre de dominio a la IP de la máquina. Para ello tenemos que editar el fichero "/etc/hosts"
![image](https://user-images.githubusercontent.com/106914229/172154103-e1f3b7a0-b352-410e-b3be-96a4fcfd2906.png)

Esto es lo que vemos ahora cuando escribimos office.paper en el navegador:

![image](https://user-images.githubusercontent.com/106914229/172154288-1fba03ae-eb23-44f3-85ac-1fe3a77e8b0a.png)

Ahora tendremos que investigar un poco la web y ver que nos puede ser útil. Tenemos lo que parece una especie de blog donde algunos usuarios han comentado. Lo primero que me llamó la atención es un panel de inicio de sesión que encontramos abajo del todo. Se me ocurrió utilizar los nombres de usuario que podemos ver en el blog y probar contraseñas débiles como por ejemplo 123456789 pero no funcionó.
Encontré algo interesante en uno de los posts:

![image](https://user-images.githubusercontent.com/106914229/172155168-bca57063-66db-4caf-9588-bdc1b7bce458.png)

Parece que la máquina nos esta dando una pista: hay "contenido secreto" que no es seguro. Al parecer la version del gestor de contenido que utiliza la página web (Wordpress 5.2.3) tiene una vulnerabilidad que nos permite ver contenido oculto tocando la URL. Es por ello, que si buscamos http://office.paper/?static=1 podremos acceder al contenido que nos interesa.

![image](https://user-images.githubusercontent.com/106914229/172162500-72637800-f072-4a26-9112-b2195c13561b.png)

Aqui nos encontramos con un enlace a un chat en el que los empleados pueden registrarse. Si accedemos directamente no nos dejará. Primero hay que añadir chat.office.paper al fichero /etc/hosts

![image](https://user-images.githubusercontent.com/106914229/172162921-bbccb3fc-8f3a-4f74-a0fe-296ed241fc47.png)

Ahora si podremos acceder a un panel de registro donde nos registraremos:
![image](https://user-images.githubusercontent.com/106914229/172163317-3def194b-ed1c-4688-9b96-73f9b95d3e57.png)
Después de habernos registrado tendremos acceso a lo que parece un chat de los empleados. No hay mucha información relevante salvo un bot que ha sido recientemente añadido y el cual nos permite tanto listar contenido de un directorio como ver el contenido de un fichero:

![image](https://user-images.githubusercontent.com/106914229/172164014-ff40ac33-4336-4f21-a627-9cd8706d5678.png)

Para poder utilizar el bot tendremos que escribirle un mensaje privado:

![image](https://user-images.githubusercontent.com/106914229/172164210-40b6dd4e-1cfa-407d-bac5-9b63ab111fd4.png)

Según lo dicho en el chat de los empleados el bot solo nos permitirá ver contenido del direcotrio "sales" (ventas en español). Sin embargo, haciendo uso de ../ podemos movernos a otros directorios. Asi es como pude leer el archivo /etc/passwd y ver que usuarios hay en el sistema:
![image](https://user-images.githubusercontent.com/106914229/172164788-04e92ecc-9404-49ea-a690-3619be2c74be.png)
![image](https://user-images.githubusercontent.com/106914229/172165081-078c6f04-0680-4e86-8075-fcafb1dd3d62.png)

Los usuarios que nos interesan son los que tienen uid de 1000 o más a excepción de root. De esta forma conocemos ya dos usuarios que pueden ser interesantes más adelante: rocketchat y dwight.
Lo siguiente que probé fue a inyectar comandos. El bot solo te deja listar directorios y ficheros (lo que viene a ser ejecutar el comando ls y el comando cat) pero podemos intentar ejecutar otros comandos de la siguiente forma:

![image](https://user-images.githubusercontent.com/106914229/172166167-45d90737-7b53-49a2-a364-01f1b285ff61.png)
Por desgracia para nosotros el bot esta preparado para evitar que se inyecten comandos. Si esto hubiera funcionado podríamos haber mandado una shell a nuestro equipo y de esa forma conectarnos al sistema, pero no ha sido así. Esto también significa que con los comandos que nos permite utilizar el bot alguna manera tiene que haber de vulnerar el sistema por lo tanto nos toca explorar todo lo posible.
Si listamos el directorio anterior veremos una carpeta que puede tener algo interesante llamada hubot.

![image](https://user-images.githubusercontent.com/106914229/172166905-f818bd36-c5cb-41bb-82e7-b5eb406b9c89.png)

Dentro de dicha carpeta encontramos lo siguiente:

![image](https://user-images.githubusercontent.com/106914229/172167022-b5496a82-1a4c-4e8f-8bf2-b1fdf632f467.png)

Debemos mirar todo lo que haya por si pudiera tener algo que nos sirva. En este caso el fichero .env contiene lo que parece un usuario y una contraseña entre otras cosas:

![image](https://user-images.githubusercontent.com/106914229/172167770-3abdb70f-f82a-4533-9d8f-218fb80e7c3d.png)

Podríamos probar a inciar sesión en el chat con esas credenciales. 

![image](https://user-images.githubusercontent.com/106914229/172168064-550d1f40-aca3-4890-9870-0cd4ed60cc81.png)

Mala suerte para nosotros. El bot no tiene permitido logearse en el chat. En este punto vino a mi mente que el puerto 22 con servicio SSH estaba abierto y que disponemos de algunos nombres de usuario asi que podriamos probar suerte y quizá la contraseña del bot nos sirva con algún usuario.

![image](https://user-images.githubusercontent.com/106914229/172169008-3f5006b5-c83f-4686-be8a-8d1cd7964983.png)

Y bualah! Estamos dentro del sistema! Al parecer la contraseña no nos sirvió para el usuario rocketchat pero si para el usuario dwight. Ahora podremos visualizar la flag del usuario.

![image](https://user-images.githubusercontent.com/106914229/172170385-9eea98e0-8ec3-449f-b532-4f6dbecabe1f.png)

Ahora solo queda escalar privilegios. En este caso voy a usar la herramienta linpeas para comprobar si hay alguna vulnerabilidad que podamos aprovechar para escalar privilegios.


Esta parte esta incompleta y pendiente de acabar


