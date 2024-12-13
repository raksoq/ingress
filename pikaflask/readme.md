docker buildx create --use
docker login
docker buildx build --platform linux/amd64,linux/arm64 -t oskarq/pikaflask:v1 --push .


