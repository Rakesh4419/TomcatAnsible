# TomcatAnsible
Installation of Tomcat, maven, Java, Git in centos and deploying the source code into tomcat server using maven as build tool
- hosts: localhost    // You have to add your servers in the hosts file of ansible
  become: true       // Giving sudo privileges to ansible user
  tasks:
    - name: Install Java     // Installign java using yum 
      yum:
        name: java-1.8.0-openjdk   // java 1.8 version
        state: present       
    - name: add tomcatuser        // addign tomcat user fot tomcat installation 
      user:
        name: tomcat
        shell: /sbin/nologin     
    - name: get_installer           // Getting tomcat using get_url module, its like wget command in linux
      get_url:
        url: https://archive.apache.org/dist/tomcat/tomcat-7/v7.0.76/bin/apache-tomcat-7.0.76.tar.gz
        dest: /tmp/    // into tmp directory 
    - name: copy
      copy:
        src: /tmp/apache-tomcat-7.0.76.tar.gz      
        dest: /usr/local/    // copying the tar archive to /usr/local directory 
        remote_src: yes      // Since the copying process is running in remote host, we have to mention remote_src as yes. 
    - name: install    
      unarchive:                              
        src: /usr/local/apache-tomcat-7.0.76.tar.gz    // untaring the tomcat package
        dest: /usr/local   
        remote_src: yes   
    - name: Change file ownership, group and permissions
      file:
        path: /usr/local/apache-tomcat-7.0.76
        owner: tomcat
        group: tomcat
        recurse: yes
        state: directory
    - name: make tomcat symbolic     // Creating symlink of tomcat installation to /usr/local/tomcat7 
      file:
        src: /usr/local/apache-tomcat-7.0.76
        dest: /usr/local/tomcat7
        owner: tomcat
        group: tomcat
        state: link
    - name: make tomcat.service
      file:
        path: /etc/systemd/system/tomcat.service   // creating systemd service of tomcat using file module 
        state:  touch
    - name: edit tomcat.service
      blockinfile:  
        dest: /etc/systemd/system/tomcat.service  //inserting multiple lines to tomcat.service file using blockinfile module.
        insertafter:
        block: |
          [Unit]
          Description = Apache Tomcat 7
          After = syslog.target network.target
          [Service]
          User = tomcat
          Group = tomcat
          Type = oneshot
          PIDFile =/usr/local/apache-tomcat-7.0.76/tomcat.pid
          RemainAfterExit = yes
          ExecStart =/usr/local/apache-tomcat-7.0.76/bin/startup.sh
          ExecStop =/usr/local/apache-tomcat-7.0.76/bin/shutdown.sh
          ExecReStart =/usr/local/apache-tomcat-7.0.76/bin/shutdown.sh;/usr/local/apache-tomcat-7.0.76/bin/startup.sh
          [Install]
          WantedBy = multi-user.target
    - name: chmod 755 tomcat.service
      file:
        path: /etc/systemd/system/tomcat.service
        mode:  0755
    - name: start tomcat
      systemd:
        name: tomcat.service
        state: started
        daemon_reload: yes
        enabled: yes
    - name: Installing git      // Installing git using yum 
      yum:
        name: git
        state: installed
    - name: Cloning project   // cloning the source code frrom git repo to the server
      ansible.builtin.git:
        repo: https://github.com/Rakesh4419/maven-web-application.git
        dest: /home/ansible/application/version{{number}}
        clone: yes
    - name: Installing maven    // installign maven using command module
      command: yum install maven -y
    - name: Building project    // building the source code using mvn command
      command: mkdir=/home/ansible/application/version{{number}}      // We are using "number" variable to maintain each versions of source code in to sperate folders
      command: chdir=/home/ansible/application/version{{number}}  mvn clean package   // Running mvn command 
    - name: Display artifact location    
      debug:
        msg: "/home/ansible/application/version{{number}}/target/maven-web-application.war"
    - name: Deleting current deployment     // Deleting current deployment in the server
      file:
        path: /usr/local/apache-tomcat-7.0.76/webapps/maven-web-application.war
        path: /usr/local/apache-tomcat-7.0.76/webapps/maven-web-application
        state: absent
    - name: Cpying artifacts   // Copying new deployment to the tomcat 
      copy:
        src: /home/ansible/application/version{{number}}/target/maven-web-application.war
        dest: /usr/local/apache-tomcat-7.0.76/webapps/
    - name: Granting ownership     
      shell: chown -R  tomcat.tomcat /usr/local/apache-tomcat-7.0.76/
    - name: Restarting tomcat    //Restarting service
      systemd:
        name: tomcat.service
        state: restarted
