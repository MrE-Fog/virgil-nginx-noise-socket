nginx_version = "nginx-1.13.0"

stage('Get nginx sources'){
    node('master'){
        clearContentUnix()
        sh "wget https://nginx.org/download/${nginx_version}.tar.gz"
        sh "tar xfz ${nginx_version}.tar.gz"
        sh "mkdir virgil-nginx-noise-socket"
        dir("$nginx_version/virgil-nginx-noise-socket"){
            checkout scm
        }
        stash excludes: "*.tar.gz", includes: '**', name: 'nginx-source'
    }
}

stage('Build'){
    node("build-docker"){
        docker.image('centos:7').inside("--user root"){
            clearContentUnix()
        }
        unstash "nginx-source"
        docker.image('centos:7').inside("--user root"){
            sh "yum install -y gcc make pcre pcre-devel pcre2 pcre2-devel openssl-devel autoconf automake flex bison git ruby ruby-devel curl libyaml-devel rpm-build"
            sh "gem install fpm"
            sh "git clone https://github.com/rweather/noise-c.git"
            sh "cd noise-c && autoreconf -i"
            sh "cd noise-c && ./configure"
            sh "cd noise-c && make"
            sh "cd noise-c && make install"
            sh "cd noise-c && mkdir noise-artifact"
            sh "cd noise-c && export DESTDIR='noise-artifact' && make install"
            sh "ls -la noise-c/include/noise/noise-artifact"
            sh "cd $nginx_version && ./configure --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --pid-path=/var/run/nginx.pid --lock-path=/var/lock/nginx.lock --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/body --http-proxy-temp-path=/var/lib/nginx/proxy --without-http_fastcgi_module --without-http_uwsgi_module --with-http_stub_status_module --with-http_gzip_static_module --with-http_ssl_module --with-debug --add-module=./virgil-nginx-noise-socket"
            sh "cd $nginx_version && make"
            sh "cd $nginx_version && mkdir nginx-artifact"
            sh "cd $nginx_version && export DESTDIR='nginx-artifact' && make install"
            sh "cp -r noise-c/include/noise/noise-artifact/* $nginx_version/nginx-artifact/"
            sh "ls -l $nginx_version/nginx-artifact"
            sh "fpm -s dir -t rpm -p ./ -m 'sk@virgilsecurity.com' --description 'Virgil Security Noise Socket nginx with plugin' \
            --rpm-use-file-permissions \
            -n 'virgil-nginx-noise-socket' -v 1.0.${BUILD_NUMBER} -C $nginx_version/nginx-artifact ./"
        }
        stash includes: "*.rpm", name: "nginx-rpm"
    }
}

stage('Deploy artifacts'){
    node('master'){
        dir("$nginx_version/virgil-nginx-noise-socket"){
            dir('ci'){
                unstash 'nginx-rpm'
            }
            sh "ansible-playbook -i ci/nginx-inventory ci/nginx-deploy.yml --extra-vars 'rpm_name=virgil-nginx-noise-socket-1.0.${BUILD_NUMBER}-1.x86_64.rpm'"
            dir('ci'){
                archiveArtifacts("*.rpm")
            }
        }
    }
}

// Utility Functions

def clearContentUnix() {
    sh "rm -fr -- *"
}

def archiveArtifacts(pattern) {
    step([$class: 'ArtifactArchiver', artifacts: pattern, fingerprint: true, onlyIfSuccessful: true])
}
