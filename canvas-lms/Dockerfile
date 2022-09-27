FROM ubuntu:20.04
LABEL maintainer="Instructure"

ARG REVISION=stable/2022-07-06
ENV RAILS_ENV development
ENV GEM_HOME /opt/canvas/.gems
ENV GEM_PATH /opt/canvas/.gems:/opt/canvas/.gem/ruby/2.7.0
ENV DEBIAN_FRONTEND noninteractive
ENV COMPILE_ASSETS_NPM_INSTALL 0
ENV PATH="$PATH:$GEM_PATH/bin"

# add nodejs and recommended ruby repos
RUN apt-get update \
    && apt-get install -y ruby ruby-dev supervisor \
        zlib1g-dev libxml2-dev libxslt1-dev libsqlite3-dev \
        libpq-dev libxmlsec1-dev curl make g++ git \
        unzip fontforge libicu-dev autoconf curl software-properties-common \
        sudo \
        build-essential \
        libldap2-dev \
        libidn11-dev \
        nano \
        locales \
        && apt-get clean && rm -Rf /var/cache/apt && rm -Rf /var/lib/apt/lists/*



RUN curl -sSL -o apache-pulsar-client-dev.deb https://archive.apache.org/dist/pulsar/pulsar-2.6.1/DEB/apache-pulsar-client-dev.deb \
  && curl -sSL -o apache-pulsar-client.deb https://archive.apache.org/dist/pulsar/pulsar-2.6.1/DEB/apache-pulsar-client.deb \
  && dpkg -i apache-pulsar-client.deb \
  && dpkg -i apache-pulsar-client-dev.deb \
  && rm apache-pulsar-client-dev.deb apache-pulsar-client.deb

RUN curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
    && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        nodejs \
        yarn \
        unzip \
        fontforge \
        && apt-get clean && rm -Rf /var/cache/apt && rm -Rf /var/lib/apt/lists/*

#Install Nginx + Passenger

# RUN sudo apt-get -y install nginx
# RUN sudo apt-get install -y dirmngr gnupg apt-transport-https ca-certificates 
# RUN sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
# RUN sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger focal main > /etc/apt/sources.list.d/passenger.list'
# RUN sudo apt-get update && sudo apt-get install -y libnginx-mod-http-passenger && rm -Rf /var/cache/apt
# RUN if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi \
#       && sudo ls /etc/nginx/conf.d/mod-http-passenger.conf
# RUN sudo service nginx restart

# Set the locale to avoid active_model_serializers bundler install failure
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

RUN groupadd -r canvasuser -g 433 && \
    adduser --uid 431 --system --gid 433 --home /opt/canvas canvasuser && \
    adduser canvasuser sudo && \
    echo '%sudo ALL=(ALL) NOPASSWD:ALL\nDefaults env_keep += "GEM_HOME RAILS_ENV REVISION LANG LANGUAGE LC_ALL"' >> /etc/sudoers

RUN sudo -u canvasuser mkdir -p $GEM_HOME \
  && sudo chmod -R 777 $GEM_HOME \
  && sudo -u canvasuser gem install --user-install bundler:2.2.19 --no-document

RUN export GEM_HOME="$(ruby -e 'puts Gem.user_dir')"
RUN export PATH="$PATH:$GEM_PATH/bin"


COPY --chown=canvasuser dbinit.sh /opt/canvas/dbinit.sh
COPY --chown=canvasuser start.sh /opt/canvas/start.sh
RUN chmod 755 /opt/canvas/*.sh

COPY supervisord.conf /etc/supervisor/supervisord.conf

RUN cd /opt/canvas \
    && sudo -u canvasuser git clone https://github.com/instructure/canvas-lms.git --branch $REVISION --single-branch

RUN chmod -R 777 /opt/canvas/canvas-lms

RUN cd /opt/canvas/canvas-lms/db/migrate \
    && rm -rf 20210823222355_change_immersive_reader_allowed_on_to_on.rb \
    && mv 20210812210129_add_singleton_column.rb 20111111214311_add_singleton_column.rb

WORKDIR /opt/canvas/canvas-lms

COPY --chown=canvasuser domain.yml config/domain.yml
COPY --chown=canvasuser database.yml config/database.yml
COPY --chown=canvasuser redis.yml config/redis.yml
COPY --chown=canvasuser cache_store.yml config/cache_store.yml
COPY --chown=canvasuser outgoing_mail.yml config/outgoing_mail.yml
COPY --chown=canvasuser public_file_server.rb config/environments/public_file_server.rb


RUN for config in amazon_s3 delayed_jobs file_store security external_migration \
       ; do cp config/$config.yml.example config/$config.yml \
       ; done

RUN sudo -u canvasuser gem install puma -v '5.6.2' --source 'https://rubygems.org/'
RUN gem install idn-ruby -v '0.1.4' --source 'https://rubygems.org/'
# RUN gem install i18n -v 1.9.1 --default

RUN sudo -u canvasuser /opt/canvas/.gem/ruby/2.7.0/bin/bundle config set --local without 'development:test' \
  &&sudo -u canvasuser /opt/canvas/.gem/ruby/2.7.0/bin/bundle config set --local without 'mysql' \
  && sudo -u canvasuser /opt/canvas/.gem/ruby/2.7.0/bin/bundle _2.2.19_ install --jobs 8

RUN sudo -u canvasuser yarn install --pure-lockfile && sudo -u canvasuser yarn cache clean
RUN sudo -u canvasuser COMPILE_ASSETS_NPM_INSTALL=0 /opt/canvas/.gem/ruby/2.7.0/bin/bundle _2.2.19_ exec rake canvas:compile_assets_dev --trace

RUN sudo -u canvasuser mkdir -p log tmp/pids public/assets public/stylesheets/compiled \
    && sudo -u canvasuser touch Gemfile.lock

RUN chown -R canvasuser: /tmp/attachment_fu/

# RUN passenger start -p 3005 -d -e production

# # canvas
EXPOSE 80 3000

HEALTHCHECK --interval=3m --start-period=5m \
   CMD /usr/local/bin/healthcheck.sh

CMD ["/opt/canvas/start.sh"]
