FROM rust:1.78.0-bookworm AS builder
COPY . /rustycan4docker
RUN cargo install --locked --path /rustycan4docker --root /rustycan4docker

FROM ubuntu:22.04
RUN apt update && apt install -y iproute2 can-utils
RUN mkdir -p /run/docker/plugins
COPY --from=builder /rustycan4docker/bin/rustycan4docker /usr/local/bin/

CMD ["/usr/local/bin/rustycan4docker"]
