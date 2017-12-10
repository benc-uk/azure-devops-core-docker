# Screenshots of the VSTS release steps

### Creating the release definition 
![screen](imgs/rel-create1.png)
![screen](imgs/rel-create2.png)

---

### Release definition - Change agent queue
![screen](imgs/rel-agent.png)

---

### Release definition - Command-line task
> Note. The arguments should be `-c "docker ps --filter \"name=mywebapp\" -q|xargs docker rm -f || true"`
![screen](imgs/rel-clean.png)

---

### Release definition - Docker run Docker image task
![screen](imgs/rel-docker.png)

