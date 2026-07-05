# KVM Training

To start the slide show:

```
docker run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --publish 3030:3030 \
  docker.io/boxcutter/slidev --remote

# Visit <http:/localhost:3030/>
```

Edit the [slides.md](./slides.md) to see the changes.

Learn more about Slidev at the [documentation](https://sli.dev/).

# Update dependencies

```bash
docker run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --entrypoint /bin/bash \
  docker.io/boxcutter/slidev

  # pin to 52.15.2 for now
  # https://github.com/slidevjs/slidev/issues/2629
  git pull
  rm -rf node_modules package-lock.json
  npm install
  git add package.json package-lock.json
  git commit -m "Sync npm lockfile"
  git push
```

# Attributions

The following projects were used for inspiration and some code was
reused for the slideware:

- https://sli.dev/ - Slidev slideware
- https://github.com/alexanderdavide/slidev-theme-academic  Pagination  - Alexander Eble
- https://github.com/jpetazzo/container.training - Idea of using markdown slideware for training - Jérôme Petazzoni
