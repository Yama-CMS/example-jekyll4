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

# Run the `jekyll serve` dev server with $PWD mounted. Run locally with `earthly +dev`.
# Note that if you exit Earthly (eg. via Ctrl+c), Earthly will immediately exit and leave the container running.
# You can manually stop it with `earthly +stop-dev` (or `docker stop earthly-dev-server`).
dev:
    ARG port=4000
    ARG container_name="earthly-dev-server"

    LOCALLY
        WITH DOCKER
            # https://github.com/envygeeks/jekyll-docker/blob/master/README.md#server
            RUN docker run --rm \
                           --volume="$PWD:/srv/jekyll:Z" \
                           --volume="$PWD/vendor/bundle:/usr/local/bundle:Z" \
                           --publish [::1]:$port:4000 \
                           --name="$container_name" \
                           jekyll/jekyll:4 \
                           jekyll serve --trace
	END

stop-dev:
    ARG container_name="earthly-dev-server"

    LOCALLY
        RUN docker stop "$container_name"


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
        swift upload --object-threads 5 --skip-identical --object-name "" $OS_CONTAINER_NAME ./$file_or_directory
        # NOTE: swift will send a `HEAD` request for every file! If there are a lot, OpenStack might start throwing `429 Too Many Requests`.
        # --skip-identical compares filesize and Etag (md5 hash). We can get both using `swift list --json`. We might want to implement a
        # custom "bulk upload" that would first list everything, then upload?


SWIFT_DELETE_BUCKET_FILES_NOT_PRESENT_LOCALLY:
    COMMAND
    ARG --required file_or_directory

    RUN --push \
        --secret OS_USERNAME \
        --secret OS_PASSWORD \
        --secret OS_AUTH_URL \
        --secret OS_AUTH_VERSION \
        --secret OS_TENANT_NAME \
        --secret OS_STORAGE_URL \
        --secret OS_CONTAINER_NAME \
        -- \
        cd $file_or_directory && \
            swift list $OS_CONTAINER_NAME | \
            perl -lne 'print if !-e' | \
            xargs --no-run-if-empty --verbose swift delete --object-threads 1 $OS_CONTAINER_NAME

        # The code above means:
        # - Get the list of files in the bucket
        # - Pipe them to a Perl one-liner to only keep files that don't exist locally. It reads like:
        #   "On every line, trim the line (-ln) and execute this code (-e): 'print $_ if file $_ does not (-e)xist'" (in perl, `$_` holds the current line and is implicit).
        # - Pipe the files-that-do-not-exist-locally list to `swift delete` via `xargs`
        #   NOTE: We use --object-threads 1 so that there is only one request made. Otherwise, swift will
        #   split all of the files into 10 threads (eg. for 50 files, it'll make 10 threads each bulk-deleting 5 files.)


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


# Copy the site that was built in `+build`, then upload it via Swift and delete files in the container that aren't in the build result.
build-and-upload:
    FROM +swift-deps
    COPY +build/build_result ./to-sync

    DO +SWIFT_UPLOAD --file_or_directory=./to-sync
    DO +SWIFT_DELETE_BUCKET_FILES_NOT_PRESENT_LOCALLY --file_or_directory=./to-sync

# Upload a robots.txt file
upload-norobots:
    FROM +swift-deps

    RUN echo 'Disallow: /' >> robots.txt
    DO +SWIFT_UPLOAD --file_or_directory=./robots.txt


deploy-production:
    BUILD +build-and-upload

deploy-preview:
    BUILD +build-and-upload
    BUILD +upload-norobots


