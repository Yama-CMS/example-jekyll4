VERSION 0.7 # https://docs.earthly.dev/docs/earthfile#version
FROM python:3

ARG --global build_dir="_site"

# Base environment for Jekyll
jekyll-base:
    FROM jekyll/jekyll:4
    COPY Gemfile Gemfile.lock .

    RUN chmod -R a+rw Gemfile.lock # `bundle install` may update the lockfile if it's out of sync.

    RUN bundle install

# Build the site
build:
    FROM +jekyll-base
    COPY . .

    RUN chmod -R a+rw Gemfile.lock
    RUN [ -d "$build_dir" ] && chmod -R a+rw $build_dir || true # chmod the build dir if it exists
    RUN [ -d ".jekyll-cache" ] && chmod -R a+rw .jekyll-cache || true # chmod the cache if it exists

    RUN bundle exec jekyll build --trace -d $build_dir

    SAVE ARTIFACT $build_dir build_result AS LOCAL $build_dir

##################
# Helper targets #
##################

# Update dependencies and save the Gemfile and gems. Run locally with `earthly +update`.
update:
    FROM +jekyll-base

    RUN bundle update

    SAVE ARTIFACT Gemfile AS LOCAL Gemfile
    SAVE ARTIFACT Gemfile.lock AS LOCAL Gemfile.lock
    SAVE ARTIFACT /usr/local/bundle/ AS LOCAL vendor/bundle

# Run the `jekyll serve` dev server from the host. Run locally with `earthly +devserver`.
devserver:
    LOCALLY
        WITH DOCKER
            # https://github.com/envygeeks/jekyll-docker/blob/master/README.md#server
            RUN docker run --rm \
                           --volume="$PWD:/srv/jekyll:Z" \
                           --volume="$PWD/vendor/bundle:/usr/local/bundle:Z" \
                           --publish [::1]:4000:4000 \
                           jekyll/jekyll:4 \
                           jekyll serve --trace
	END


################
# Swift upload #
################

# Base environment for Swift
swift-deps:
    FROM python:3
    RUN pip install python-keystoneclient python-swiftclient


# User-Defined Command to run Swift
SWIFT_UPLOAD:
    COMMAND
    ARG --required file_or_directory
    RUN --secret OS_USERNAME \
        --secret OS_PASSWORD \
        --secret OS_AUTH_URL \
        --secret OS_AUTH_VERSION \
        --secret OS_TENANT_NAME \
        --secret OS_STORAGE_URL \
        --secret OS_CONTAINER_NAME \
        --push \
        -- \
        swift upload --changed --object-name "" $OS_CONTAINER_NAME ./$file_or_directory

SWIFT_EMPTY_BUCKET:
    COMMAND
    RUN --push \
        --secret OS_USERNAME \
        --secret OS_PASSWORD \
        --secret OS_AUTH_URL \
        --secret OS_AUTH_VERSION \
        --secret OS_TENANT_NAME \
        --secret OS_STORAGE_URL \
        --secret OS_CONTAINER_NAME \
        -- \
        swift list $OS_CONTAINER_NAME | xargs --no-run-if-empty --verbose swift delete $OS_CONTAINER_NAME


# Run `swift list` on the container
swift-list:
    FROM +swift-deps

    RUN --push \
        --secret OS_USERNAME \
        --secret OS_PASSWORD \
        --secret OS_AUTH_URL \
        --secret OS_AUTH_VERSION \
        --secret OS_TENANT_NAME \
        --secret OS_STORAGE_URL \
        --secret OS_CONTAINER_NAME \
        -- \
        swift list $OS_CONTAINER_NAME


# Copy the site that was built in `+build`, then upload it via Swift
upload-site:
    FROM +swift-deps
    COPY +build/build_result ./to-upload

    DO +SWIFT_EMPTY_BUCKET
    DO +SWIFT_UPLOAD --file_or_directory=./to-upload


# Upload a robots.txt file
upload-norobots:
    FROM +swift-deps

    RUN echo 'Disallow: /' >> robots.txt
    DO +SWIFT_UPLOAD --file_or_directory=./robots.txt

