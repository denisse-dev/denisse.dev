---
title: Manteniendo paquetes del AUR de Arch Linux
description: Cómo mantengo paquetes para el AUR, el flujo que sigo al mantener paquetes y las herramientas que uso
date: 2021-12-14
slug: manteniendo-paquetes-del-aur-de-arch-linux
image: magda-ehlers-selective-focus-photography-of-a-gray-penguin.jpg
categories:
    - linux
    - emacs
tags:
    - aur
    - arch
    - emacs
---

## El AUR

Una de las cosas que más me gustan de usar Arch Linux es el [AUR](https://wiki.archlinux.org/title/Arch_User_Repository_(Espa%C3%B1ol)), un repositorio comunitario con miles de [PKGBUILDs](https://wiki.archlinux.org/title/PKGBUILD_(Espa%C3%B1ol)) que permiten instalar y actualizar programas fácilmente y favorecen la colaboración comunitaria.

## Mi entorno

Al editar `PKGBUILD`s uso el modo [pkgbuild-mode](https://melpa.org/#/pkgbuild-mode) en Emacs para tener resaltado de sintaxis, validación de errores de sintaxis y autocompletado.

Con el siguiente snippet habilito la detección de archivos PKGBUILD para automáticamente iniciar `pkgbuild-mode`:

```emacs-lisp
(autoload 'pkgbuild-mode "pkgbuild-mode.el" "PKGBUILD mode." t)
 (setq auto-mode-alist (append '(("/PKGBUILD$" . pkgbuild-mode))
			  auto-mode-alist))
```

Recomiendo instalar y configurar los siguientes paquetes:

0. [paru](https://aur.archlinux.org/packages/paru/): [Ayudante de AUR](https://wiki.archlinux.org/title/AUR_helpers_(Espa%C3%B1ol))
1. [namcap](https://wiki.archlinux.org/title/Namcap): Para detectar errores en el PKGBUILD.
2. [aurpublish](https://github.com/eli-schwartz/aurpublish): Framework para interactuar con el AUR.

## Tipos de paquetes

En el AUR podemos encontrar 3 tipos de paquetes, usaré el paquete [benthos](https://www.benthos.dev/) como ejemplo:

0. Paquete [benthos](https://aur.archlinux.org/packages/benthos/) en el AUR

   El paquete `benthos` descarga y compila el código fuente de un release de benthos que se identifica con un tag de Git en upstream.

1. Paquete [benthos-bin](https://aur.archlinux.org/packages/benthos-bin/) en el AUR

   El paquete `benthos-bin` descarga un binario para las arquitecturas soportadas un release de benthos en upstream.

2. Paquete [benthos-git](https://aur.archlinux.org/packages/benthos-git/) en el AUR

   El paquete `benthos-git` descarga y compila el código fuente del último commit en upstream.

Aunque los 3 paquetes tienen un nombre diferente en el AUR el paquete que instalan debe llamarse `benthos`y tener conflicto con `benthos` para evitar que pacman instale diferentes versiones de un mismo paquete al mismo tiempo, así se puede hacer en un `PKGBUILD`:

```bash
pkgname=benthos
provides=($pkgname)
conflicts=($pkgname)
```

## Contribuyendo al AUR

Una de las primeras maneras de contribuir a Arch Linux y el primer paso para ser [Trusted User](https://wiki.archlinux.org/title/Trusted_Users_(Espa%C3%B1ol)) es mantener paquetes para el AUR.

Sigo los siguientes pasos al empaquetar software para el AUR.

0. [Escoger lo que voy a empaquetar](#escoger-lo-que-voy-a-empaquetar)
1. [Lo que un PKGBUILD debe instalar y cómo lo debe hacer](#lo-que-un-pkgbuild-debe-instalar-y-cómo-lo-debe-hacer)
2. [Validar el PKGBUILD](#validar-el-pkgbuild)
3. [Actualizar checksums](#actualizar-checksums)
4. [Construir el paquete](#construir-el-paquete)
5. [Instalar el paquete](#instalar-el-paquete)
6. [Generar .SRCINFO](#generar-srcinfo)
7. [Publicar el paquete](#publicar-el-paquete)

### Escoger lo que voy a empaquetar

El primer paso es verificar que lo que queremos empaquetar no esté en el AUR para evitar duplicados.

Para inicializar el repositorio de un paquete nuevo:

```bash
$ aurpublish setup
```

Para clonar el repositorio de un [paquete huérfano](https://aur.archlinux.org/packages/?SB=n&do_Orphans=Orphans):

```bash
$ aurpublish -p PACKAGE
```

### Lo que un PKGBUILD debe instalar y cómo lo debe hacer

Un PKGBUILD debe instalar lo siguiente:

0. Instalar el paquete.
1. Instalar la licencia el paquete usa una licencia que no se encuentra en el archivo [licenses](https://archlinux.org/packages/core/any/licenses/) (ej. [MIT](https://en.wikipedia.org/wiki/MIT_License)).
2. Instalar la configuración por defecto si en upstream hay una.
3. Instalar servicios si en upstream hay algunos.

Y debe hacerlo siguiendo la [_Guía para paquetes de Arch_](https://wiki.archlinux.org/title/Arch_package_guidelines_(Espa%C3%B1ol)) y la guía específica para el lenguaje que estás empaquetando.

En el [PKGBUILD](https://github.com/da-edra/pkgbuilds/blob/trunk/benthos/PKGBUILD) que escribí para `benthos` seguí la _Guía para paquetes de Arch_ y la [_Guía para paquetes de Go_](https://wiki.archlinux.org/title/Go_package_guidelines) para obtener ventajas en seguridad como [PIE](https://access.redhat.com/blogs/766093/posts/1975793) y mejorar la [reproducibilidad del paquete](https://wiki.archlinux.org/title/Reproducible_Builds):

```shell
pkgname=benthos
pkgdesc='Declarative stream processing for mundane tasks and data engineering.'
arch=(aarch64 armv5h armv6h armv7h x86_64)
url='https://benthos.dev'
_url='https://github.com/Jeffail/benthos'
_branch='master'
pkgver=3.60.1
pkgrel=1
license=('MIT')
makedepends=(go)
source=("${pkgname}-${pkgver}.tar.gz::${_url}/archive/v${pkgver}.tar.gz")
b2sums=('8858cb29c12fbea787b1df82e887ca20b9150cdada6c0ea44af46f7c46f430fc2aa1360d9568a4035ba81bb5ac1d1e0d50e1df4cbb04b252e69c491006823171')
provides=($pkgname)
conflicts=($pkgname)

build() {
  cd "${pkgname}-${pkgver}"

  export CGO_LDFLAGS="${LDFLAGS}"
  export CGO_CFLAGS="${CFLAGS}"
  export CGO_CPPFLAGS="${CPPFLAGS}"
  export CGO_CXXFLAGS="${CXXFLAGS}"
  export GOFLAGS="-buildmode=pie -trimpath -ldflags=-linkmode=external -mod=readonly -modcacherw"

  go build -o benthos cmd/benthos/main.go
}

check() {
  cd "${pkgname}-${pkgver}"
  go test ./...
}

package() {
  cd "${pkgname}-${pkgver}"
  install -Dm755 $pkgname "$pkgdir"/usr/bin/$pkgname
  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
```

### Validar el PKGBUILD

Uso `namcap` para buscar errores comunes en el PKGBUILD:

```bash
$ namcap PKGBUILD
```

La configuración por defecto del archivo [makepkg.conf](https://man.archlinux.org/man/makepkg.conf.5.en) ejecuta la función `check()` si se encuentra en el `PKGBUILD`. En la función `check()` se pueden agregar diferentes validaciones adicionales como ejecutar los tests que el paquete contiene upstream.

Check también se puede ejecutar manualmente:

```bash
$ makepkg --check PKGBUILD
```

### Actualizar checksums

El `PKGBUILD` de paquetes de tipo `PACKAGE` y `PACKAGE-bin` debe tener checksums para validar la [integridad del paquete](https://wiki.archlinux.org/title/PKGBUILD#Integrity). Usar los hashes que se encuentran en upstream garantiza la integridad del paquete desde su publicación hasta su instalación, ej:

```bash
b2sums=('79586b8592a246ddb9a75752329e94bcc8d8924dcbe2eb2c7bdd1a11981e4ee39abcea86fb7b76e8c54dc8dd0f20d8b5d4b5f63025380f1ed9efbcca8c9b0bb7')
```

En el caso de `PACKAGE-git` se debe saltar la validación de checksums.

```
b2sums=(SKIP)
```

### Construir el paquete

Los [paquetes que mantengo](https://aur.archlinux.org/packages/?O=0&SeB=M&K=denisse&outdated=&SB=n&SO=a&PP=50&do_Search=Go) están organizados en carpetas con el nombre del paquete y contienen un `PKGBUILD` y un `.SRCINFO`:

```
.
├── benthos
│  ├──  .SRCINFO
│  └──  PKGBUILD
├── benthos-bin
│  ├──  .SRCINFO
│  └──  PKGBUILD
└── benthos-git
   ├──  .SRCINFO
   └──  PKGBUILD
```

Construyo el paquete con `makepkg`:

```bash
$ makepkg --clean --syncdeps --force
```

`--clean` para limpiar el directorio después de construir el paquete

`--syncdeps` para instalar dependencias faltantes

`--force` para sobreescribir un paquete ya construido.

### Instalar el paquete

Instalo el paquete con `makepkg`:

```bash
$ makepkg --install
```

Y realizo las siguientes validaciones:

0. El paquete funciona como se espera.
1. Si se necesitan licencias éstas están instaladas.
2. Si hay una configuración por defecto ésta está instalada y funciona.
3. Si en upstream hay demonios estos están instalados y funcionan.

### Generar .SRCINFO

Con `makepkg` creo al archivo [.SRCINFO](https://wiki.archlinux.org/title/.SRCINFO) que contiene metadatos organizados en pares de `llave = valor` que es más fácil de parsear y seguro de parsear que un PKGBUILD.

```bash
$ makepkg --printsrcinfo > .SRCINFO
```

### Publicar el paquete

El último paso es usar `aurpublish` para publicar el paquete:

```bash
$ aurpublish PACKAGE
```

De esta forma nuestro paquete estará publicado en el AUR. 😊

## Makefile para publicar paquetes

Para facilitar el publicar paquetes uso un [Makefile](https://github.com/da-edra/pkgbuilds/blob/trunk/Makefile) que contiene las instrucciones descritas en la sección anterior:

```makefile
.PHONY: help check-pkg update-checksums build-pkg install-pkg update-srcinfo update-pkg publish
.DEFAULT_GOAL := help

help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

check-pkg: ## check PKGBUILDs for common packaging mistakes.
	namcap $(pkg)/PKGBUILD
	echo "Done checking the package"

update-checksums: ## update PKGBUILD checksums
	updpkgsums $(pkg)/PKGBUILD
	echo "Done updating checksums"

build-pkg: ## build the PKGBUILD
	cd $(pkg) && \
	makepkg --clean --syncdeps -f && \
	echo "Done building the PKGBUILD"

install-pkg: ## install the package after building it
	cd $(pkg) && \
	makepkg --install
	echo "Done installing the package"

update-srcinfo: ## print SRCINFO into a file
	cd $(pkg) && \
	makepkg --printsrcinfo > .SRCINFO && \
	echo "Done updating the .SRCINFO"

update-pkg: check-pkg update-checksums build-pkg install-pkg update-srcinfo ## update and upload the package to the AUR

publish: ## publish the PKGBUILD to the AUR
	aurpublish $(pkg)
```

Actualizando un PKGBUILD:

```bash
make update-pkg pkg=PACKAGE`
```

Publicando un PKGBUILD:

```bash
make publish pkg=PACKAGE`
```
