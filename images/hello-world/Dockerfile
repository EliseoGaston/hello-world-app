FROM nginx:alpine 

# Copiar el archivo index.html a la posicion por defecto de nginx 
COPY index.html /usr/share/nginx/html/index.html

# le asignamos los permisos sobre el archivo copiado al usuario nginx por ejecutar el copy como root
RUN chown -R nginx /usr/share/nginx/html

EXPOSE 80
