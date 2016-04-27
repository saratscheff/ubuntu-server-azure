1. **Crear máquina virtual en http://manage.windowsazure.com**
2. **Crear usuario theteam en http://portal.azure.com**
  * Mediante Reset-password, se escribe user theteam y contraseña deseada
3. **Configurar ENDPOINTS (Puertos)**
  
  | Name          | Protocol      | Public Port  | Private Port  |
  | ------------- |---------------| -------------| --------------|
  | SSH           | TCP           | 22           | 22            |
  | HTTP          | TCP           | 80           | 3000          |
  | PostgreSQL    | TCP           | 5432         | 5432          |

4. **Conectar a la máquina virtual**
  * Configurar la contraseña deseada anterior como ssh public key o en su defecto utilizar el siguiente código para evitar colocar la contraseña en cada conexión futura mediante ssh:
  * `ssh-copy-id theteam@apuntate.cloudapp.net`
  
  1. `ssh theteam@apuntate.cloudapp.net`
  2. `su theteam` para cambiar de usuario
  3. `sudo apt-get update` para comenzar la actualización
5. **Corregir locale**
  1. `sudo nano /etc/environment`
  2. Agregar linea `LC_ALL="en_US.UTF-8"` al archivo
  3. Reiniciar máquina virtual en http://manage.windowsazure.com
  4. `sudo dpkg-reconfigure locales` para verificar que se logró correctamente
6. **Instalar ruby mediante .rbenv**
  * `sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev` [LENTO]

  1. `cd`
  2. `git clone git://github.com/sstephenson/rbenv.git .rbenv`
  3. `echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc`
  4. `echo 'eval "$(rbenv init -)"' >> ~/.bashrc`
  5. `exec $SHELL`
  6. `git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build`
  7. `echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc`
  8. `exec $SHELL`
  9. `git clone https://github.com/sstephenson/rbenv-gem-rehash.git ~/.rbenv/plugins/rbenv-gem-rehash`
  10. `rbenv install 2.2.0` [MUUUUY LENTO - coffee time?]
  11. `rbenv global 2.2.0`
  12. `ruby -v` Para verificar instalación
7. **Evitar documentaciones al instalar gemas**
  1. echo "gem: --no-ri --no-rdoc" > ~/.gemrc
8. **Instalar Nginx**
  1. `sudo apt-get update`
  2. `sudo apt-get install curl git-core nginx -y`
9. **Instalar PostgreSQL 9.3**
  1. `psql -v` Verificar que no está instalado
  2. `sudo apt-get install postgresql-9.3`
  3. `sudo -u postgres psql postgres` Para verificar instalación. Para salir: `\q`
10. **Configurar PostgreSQL**
  1. `sudo -u postgres createuser --superuser theteam`
  2. `sudo -u postgres psql`
  3. postgres=# `\password theteam` para setear contraseña
  
  * Crear base de datos production??
11. **Instalar gemas de rails y bundler**
  1. `gem install rails` [LENTO]
  2. `gem install bundler`
12. **Setear conexión ssh a la repo**
  1. `ssh -T git@github.com` - Handshake (Está bien recibir un `Permission denied (publickey)`)
  2. `ssh-keygen -t rsa` (Dejar todas las opciones en blanco)
  3. `nano ~/.ssh/id_rsa.pub` copiar texto al ssh keys de la cuenta en github
13. **clonar app**
  1. `git clone #ssh-repo-url`
14. **Preparar App**
  1. utilizar `cd theteam` para poscicionarse en el directorio de la app clonada
  2. `sudo apt-get install libpq-dev` Para poder instalar gema postgres
  3. `bundle install`
  4. `sudo apt-get install nodejs` Para poder realizar varias funciones de rails
15. **Confirmar datos y configuraciones de puma y pg**
  1. Verificar que las gemas puma y pg están en Gemfile
  2. Verificar variables para username y password en production en database.yml
  3. Configurar puma con texto de config/puma.rb con el siguiente texto (REEMPLAZAR workers 1 POR EL NUMERO DE NUCLEOS del server!!):
  ```
  # Change to match your CPU core count
workers 1

# Min and Max threads per worker
threads 1, 6

app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"

# Default to production
rails_env = ENV['RAILS_ENV'] || "production"
environment rails_env

# Set up socket location
bind "unix://#{shared_dir}/sockets/puma.sock"

# Logging
stdout_redirect "#{shared_dir}/log/puma.stdout.log", "#{shared_dir}/log/puma.stderr.log", true

# Set master PID and state locations
pidfile "#{shared_dir}/pids/puma.pid"
state_path "#{shared_dir}/pids/puma.state"
activate_control_app

on_worker_boot do
  require "active_record"
  ActiveRecord::Base.connection.disconnect! rescue ActiveRecord::ConnectionNotEstablished
  ActiveRecord::Base.establish_connection(YAML.load_file("#{app_dir}/config/database.yml")[rails_env])
end
```
16. **Obtener secret key del developer computer (?)**
  1. EN LA COMPUTADORA DEVELOPER ejecutar en el directorio de la app `rake secret` y guardar key
17. **Create Puma Upstart script**
  1. `cd ~`
  2. `wget https://raw.githubusercontent.com/puma/puma/master/tools/jungle/upstart/puma-manager.conf`
  3. `wget https://raw.githubusercontent.com/puma/puma/master/tools/jungle/upstart/puma.conf`
  4. `nano puma.conf`
  5. Reemplazar: 

     ```
        setuid *apps*
        setgid *apps*
     ```
     Por:
     ```
        setuid theteam
        setgid theteam
     ```
     Y EN EL MISMO ARCHIVO AGREGAR justo debajo de `exec /bin/bash <<'EOT'` el siguiente código (REEMPLAZAR CONTRASEÑA Y SECRET KEY):
     ```
     export THETEAM_DATABASE_USER='theteam'
     export THETEAM_DATABASE_PASSWORD='LA_CONTRASEÑA!'
     export SECRET_KEY_BASE='LA_SECRET_KEY_GENERADA_ARRIBA!'
     ```
     Salir y guardar el archivo y sus cambios.
18. **Copiar los scripts al directorio init**
  1. `sudo cp puma.conf puma-manager.conf /etc/init`
19. **Crear y configurar archivo etc/puma.conf**
  1. `sudo nano /etc/puma.conf`
  2. Escribir `/home/theteam/theteam`
  3. Salir y guardar cambios
20. **Configurar Nginx**
  1. `sudo nano /etc/nginx/sites-available/default`
  2. Reemplazar/borrar todo y escribir: (Notar texto específico *[...]theteam/theteam/[...]* en dos casos)

     ```
     upstream app {
     # Path to Puma SOCK file, as defined previously
     server unix:/home/theteam/theteam/shared/sockets/puma.sock fail_timeout=0;
     }
     
     server {
         listen 80;
         server_name localhost;
     
         root /home/theteam/theteam/public;
     
         try_files $uri/index.html $uri @app;
     
         location @app {
             proxy_pass http://app;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header Host $http_host;
             proxy_redirect off;
         }
     
         error_page 500 502 503 504 /500.html;
         client_max_body_size 4G;
         keepalive_timeout 10;
     }
     ```
  3. Guardar y salir
21. Preparar producción git
  1. `mkdir ~/theteam_production`
  2. `cd ~/theteam_production`
  3. `git init --bare`
22. **Git hook script**
  1. `nano hooks/post-receive`
  2. Escribir en el archivo (CAMBIAR LA CONTRASEÑA!!):

     ````
     #!/bin/bash
     
     GIT_DIR=/home/theteam/theteam_production
     WORK_TREE=/home/theteam/theteam
     export THETEAM_DATABASE_USER='theteam'
     export THETEAM_DATABASE_PASSWORD='LA_CONTRASEÑA!!'
     
     export RAILS_ENV=production
     . ~/.bash_profile
     
     while read oldrev newrev ref
     do
         if [[ $ref =~ .*/master$ ]];
         then
             echo "Master ref received.  Deploying master branch to production..."
             mkdir -p $WORK_TREE
             git --work-tree=$WORK_TREE --git-dir=$GIT_DIR checkout -f
             mkdir -p $WORK_TREE/shared/pids $WORK_TREE/shared/sockets $WORK_TREE/shared/log
     
             # start deploy tasks
             cd $WORK_TREE
             $HOME/.rbenv/shims/bundle install
             $HOME/.rbenv/shims/rake db:create
             $HOME/.rbenv/shims/rake db:migrate
             $HOME/.rbenv/shims/rake assets:precompile
             sudo service puma-manager restart
             sudo service nginx restart
             # end deploy tasks
             echo "Git hooks deploy complete"
         else
             echo "Ref $ref successfully received.  Doing nothing: only the master branch may be deployed on this server."
         fi
     done
     ```
  3. Guardar y salir
23. **Hacer el script ejecutable**
  1. `chmod +x hooks/post-receive`
  2. `sudo sh -c 'echo "theteam ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-deploy'` para no requerir contraseña con el sudo el user *theteam* al ejecutar comandos dentro del script (O cualquier comando)

**LISTO!!**
En developer machine:

24. `git remote add production theteam@apuntate.cloudapp.net:theteam_production`
25. `git push production master`
