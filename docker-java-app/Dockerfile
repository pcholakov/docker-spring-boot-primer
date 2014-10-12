FROM docker-java-base
ENV HOME /root
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh
CMD ["/sbin/my_init"]
EXPOSE 8080
COPY ../app/build/libs/gs-spring-boot-0.1.0.jar /opt/
ENTRYPOINT ["/usr/bin/java", "-jar", "/opt/gs-spring-boot-0.1.0.jar"]
