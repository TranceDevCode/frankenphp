# PHP Apps As Standalone Binaries

FrankenPHP has the ability to embed the source code and assets of PHP applications in a static, self-contained binary.

Thanks to this feature, PHP applications can be distributed as standalone binaries that include the application itself, the PHP interpreter and Caddy, a production-level web server.

## Preparing Your App

Before creating the self-contained binary be sure that your app is ready for embedding.

For instance you likely want to:

* Install the production dependencies of the app
* Dump the autoloader
* Enable the production mode of your application (if any)
* Strip uneeded files such as `.git` or tests to reduce the size of your final binary

For instance, for a Symfony app, you can use the following commands:

```console
# Export the project to get rid of .git/, etc
mkdir $TMPDIR/my-prepared-app
git archive HEAD | tar -x -C $TMPDIR/my-prepared-app
cd $TMPDIR/my-prepared-app

# Set proper environment variables
echo APP_ENV=prod > .env.local
echo APP_DEBUG=0 >> .env.local

# Remove the tests
rm -Rf tests/

# Install the dependencies
composer install --ignore-platform-reqs --no-dev -a

# Optimize .env
composer dump-env prod
```

## Creating a Linux Binary

The easiest way to create a Linux binary is to use the Docker-based builder we provide.

1. Create a file named `static-build.Dockerfile` in the repository of your prepared app:

    ```dockerfile
    FROM --platform=linux/amd64 dunglas/frankenphp:static-builder

    # Copy your app
    WORKDIR /go/src/app/dist/app
    COPY . .

    # Build the static binary, be sure to select only the PHP extensions you want
    WORKDIR /go/src/app/
    RUN EMBED=dist/app/ \
        PHP_EXTENSIONS=ctype,iconv,pdo_sqlite \
        ./build-static.sh
    ```

2. Build:

    ```console
    docker build -t static-app -f static-build.Dockerfile .
    ```

3. Extract the binary

    ```console
    docker cp $(docker create --name static-app-tmp static-app):/go/src/app/dist/frankenphp-linux-x86_64 my-app ; docker rm static-app-tmp
    ```

The resulting binary is the file named `my-app` in the current directory.

## Creating a Binary for Other OSes

If you don't want to use Docker, or want to build a macOS binary, use the shell script we provide:

```console
git clone https://github.com/dunglas/frankenphp
cd frankenphp
RUN EMBED=/path/to/your/app \
    PHP_EXTENSIONS=ctype,iconv,pdo_sqlite \
    ./build-static.sh
```

The resulting binary is the file named `frankenphp-<os>-<arch>` in the `dist/` directory.

## Using The Binary

This is it! The `my-app` file contains your self-contained app!

To start the web app run:

```console
./my-app php-server
```

If your app contains a [worker script](worker.md), start the worker with something like:

```console
./my-app php-server --worker public/index.php
```

You can also run the PHP CLI scripts embedded in your binary:

```console
./my-app php-cli bin/console
```

## Customizing The Build

[Read the static build documentation](static.md) to see how to customize the binary (extensions, PHP version...).

## Distributing The Binary

The created binary isn't compressed.
To reduce the size of the file before sending it, you can compress it.

We recommend `xz`.