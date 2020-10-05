# Concourse on macOS

How to deploy and run Concourse on macOS natively

Used version 5.2.4 of Concourse becouse from v5.2.4 to v6 I got an HTTP error when the web page is loadded.

```
CONCOURSE_HOME=__set_home_dir__
CONCOURSE_BIND_PORT=8080

# Install Concourse (supported version)
# Install Postgres
# Install Docker https://docs.docker.com/docker-for-mac/install
brew install postgres
wget https://github.com/concourse/concourse/releases/download/v5.2.4/concourse-5.2.4-darwin-amd64.tgz

# Create directories
mkdir -p $CONCOURSE_HOME
mkdir $CONCOURSE_HOME/worker_macos
cd $CONCOURSE_HOME

# Create Database
pg_ctl -D /usr/local/var/postgres stop
rm -rf /usr/local/var/postgres
initdb /usr/local/var/postgres
pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log start
createdb atc && createuser concourse --pwprompt

# Create Keys
ssh-keygen -t rsa -q -N '' -f session_signing_key -m pem
ssh-keygen -t rsa -q -N '' -f tsa_host_key -m pem
ssh-keygen -t rsa -q -N '' -f worker_macos_key -m pem
ssh-keygen -t rsa -q -N '' -f worker_linux_key -m pem

echo worker_macos_key >> authorized_worker_keys
echo worker_linux_key >> authorized_worker_keys

# Start web
concourse web \
	--bind-port=$CONCOURSE_BIND_PORT \
	--add-local-user=admin:admin \
	--main-team-local-user=admin \
	--session-signing-key=$CONCOURSE_HOME/session_signing_key \
	--tsa-host-key=$CONCOURSE_HOME/tsa_host_key \
	--tsa-authorized-keys $CONCOURSE_HOME/authorized_worker_keys \
	--postgres-host=127.0.0.1 \
	--postgres-port=5432 \
	--postgres-user=concourse \
	--postgres-password=concourse

# Start macos worker
sudo concourse worker \
	--work-dir $CONCOURSE_HOME/worker_macos \
	--tsa-host 127.0.0.1:2222 \
	--tsa-public-key $CONCOURSE_HOME/tsa_host_key.pub \
	--tsa-worker-private-key $CONCOURSE_HOME/worker_macos_key

# Install and run linux worker on a Docker container
# Create docker-compose-linux-worker.yml (see assets folder)
# Replace home and web ip address
# Creante and run container
docker-compose -f docker-compose-linux-worker.yml up -d

```

