# Docker Chain Commands

---

## What is Command Chaining?

Running multiple Docker commands together in one line using shell operators.
Instead of running commands one by one, you chain them to save time and avoid mistakes.

---

## Shell Operators

| Operator | Meaning                                         |
|----------|-------------------------------------------------|
| `&&`     | Run next command **only if** previous succeeded |
| `\|\|`   | Run next command **only if** previous failed    |
| `;`      | Run next command **no matter what**             |
| `\|`     | Pass output of one command as input to next     |
| `$()`    | Use output of a command as an argument          |

---

## Quick Reference

|                               | Command                                                                  |
|-------------------------------|--------------------------------------------------------------------------|
| Build and run                 | `docker build -t img . && docker run --rm img`                           |
| Stop and remove               | `docker stop c && docker rm c`                                           |
| Remove all stopped containers | `docker ps -aq \| xargs docker rm`                                       |
| Remove dangling images        | `docker images -f dangling=true -q \| xargs docker rmi`                  |
| Build + tag + push            | `docker build -t img . && docker tag img reg/img && docker push reg/img` |
| Start + follow logs           | `docker start c && docker logs -f c`                                     |
| Full system cleanup           | `docker system prune -af`                                                |

---
