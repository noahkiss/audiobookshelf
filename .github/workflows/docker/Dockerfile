# syntax=docker/dockerfile:1.6-labs

ARG GIT_TAG=${GIT_TAG:-master}
ARG BASE_IMAGE=${BASE_IMAGE:-ghcr.io/linuxserver/baseimage-alpine:3.18}

### STAGE 0: Get deps ###
FROM node:16-alpine AS deps

ARG GIT_TAG

ADD https://github.com/advplyr/audiobookshelf.git#${GIT_TAG} audiobookshelf

WORKDIR /client

RUN cp /audiobookshelf/client/package* ./

RUN npm ci

### STAGE 1: Build client ###
FROM node:16-alpine AS build

COPY ./.github/workflows/docker/root /final
COPY --from=sandreas/tone:v0.1.5 /usr/local/bin/tone /final/usr/local/bin/
COPY --from=ceramicwhite/static-ffmpeg:latest /ffmpeg /ffprobe /final/usr/bin/
COPY --from=deps /audiobookshelf/index.js /audiobookshelf/package* /audiobookshelf/readme.md /audiobookshelf/LICENSE /audiobookshelf/Dockerfile /final/app/
COPY --from=deps /audiobookshelf/server /final/app/server

COPY --from=deps /audiobookshelf/client /client
COPY --from=deps /client/node_modules /client/node_modules

RUN cd /client && \
    npm run generate

RUN mkdir -p /final/app/client
RUN mv /client/dist /final/app/client/

### STAGE 2: Server ###
FROM ${BASE_IMAGE} AS runner

ARG \
    BUILDPLATFORM \
    BUILDOS \
    BUILDARCH \
    TARGETPLATFORM \
    TARGETOS \
    TARGETARCH \
    TARGETOS \
    GIT_TAG \
    BASE_IMAGE

ENV \
    NODE_ENV=${NODE_ENV:-production} \
    TZ=${TZ:-UTC} \
    FFMPEG_PATH=/usr/bin/ffmpeg \
    FFPROBE_PATH=/usr/bin/ffprobe \
    PORT=${PORT:-80} \
    PUID=${PUID:-1000} \
    PGID=${PGID:-1000}

COPY --from=build /final/ /

RUN \
    apk --update --upgrade --no-cache add \
        ca-certificates \
        tzdata \
        nodejs \
        npm \
        jq && \
    update-ca-certificates && \
    cd /app && \
    npm ci --omit-dev && \
    echo -e "\n\n[√] Finished Docker build successfully. Saving build summary in: /VERSION.txt\n" && \
    ( \
    echo -e "     **** BUILD INFO ****\n" && \
    echo -e "$(date +"%Y-%m-%d %H:%M:%S %s") ${TZ}\n" && \
    echo -e "Builder:     ${BUILDOS}/${BUILDARCH}\n" && \
    echo -e "Final Image: ${BASE_IMAGE}" && \
    echo -e "OS/Arch:     $(uname -m) (${TARGETOS}/${TARGETARCH})\n" && \ 
    echo -e "App:         v$(cat /app/package.json | jq -r '.version')" && \
    echo -e "Tag:         ${GIT_TAG}" && \ 
    echo -e "Commit:      $(curl -s https://api.github.com/repos/advplyr/audiobookshelf/commits/${GIT_TAG} | jq -r '.sha' | cut -c 1-7)\n" && \
    echo -e "Node:        $(node --version)" && \
    echo -e "NPM:         v$(npm --version)" && \
    echo -e "Tone:        v$(tone --version)" && \
    echo -e "FFMPEG:      v$(ffmpeg -version | grep ffmpeg | cut -d' ' -f3)\n\n" \
    ) | tee -a /VERSION.txt

EXPOSE 80