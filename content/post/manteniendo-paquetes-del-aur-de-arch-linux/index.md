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

Una de las cosas que más me gustan de usar [Arch Linux](https://archlinux.org/) es el [AUR](https://wiki.archlinux.org/title/Arch_User_Repository_(Espa%C3%B1ol)), un repositorio comunitario con miles de [PKGBUILD](https://wiki.archlinux.org/title/PKGBUILD_(Espa%C3%B1ol)) que permiten instalar y actualizar programas fácilmente y que favorece la colaboración comunitaria.

## Mi entorno

Al editar un **PKGBUILD** uso el modo [pkgbuild-mode](https://melpa.org/#/pkgbuild-mode) en [Emacs](https://wiki.archlinux.org/title/Emacs_(Espa%C3%B1ol)) para tener resaltado de sintaxis, validación de sintaxis y autocompletado.

Con el siguiente snippet de [Emacs Lisp](https://www.gnu.org/software/emacs/manual/html_node/elisp/index.html) habilito la detección de archivos **PKGBUILD** para automáticamente iniciar **pkgbuild-mode** en Emacs:

```emacs-lisp
(autoload 'pkgbuild-mode "pkgbuild-mode.el" "PKGBUILD mode." t)
 (setq auto-mode-alist (append '(("/PKGBUILD$" . pkgbuild-mode))
			  auto-mode-alist))
```

Recomiendo instalar y configurar los siguientes paquetes:

0. [paru](https://aur.archlinux.org/packages/paru/): [Ayudante de AUR](https://wiki.archlinux.org/title/AUR_helpers_(Espa%C3%B1ol))
1. [namcap](https://archlinux.org/packages/extra/any/namcap/): Para detectar errores en archivos PKGBUILD.
2. [aurpublish](https://archlinux.org/packages/community/any/aurpublish/): Framework para interactuar con el AUR.

## Tipos de paquetes

En el **AUR** podemos encontrar 3 tipos de paquetes que podemos identificar como **PAQUETE**, **PAQUETE-bin** y **PAQUETE-git**. Usaré el paquete [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/) como ejemplo:

0. [oauth2-proxy](https://aur.archlinux.org/packages/oauth2-proxy/):

   El paquete **oauth2-proxy** descarga y compila el código fuente de un release de **oauth2-proxy** que se identifica con un tag de [Git](https://git-scm.com/) en upstream.

   Ventajas:

   - Provee una versión considerada estable del paquete
   - El binario se puede compilar para múltiples arquitecturas
   - El binario se puede optimizar para tu equipo

   Desventajas:

   - Compilarlo puede ser tardado en ciertos equipos

1. [oauth2-proxy-bin](https://aur.archlinux.org/packages/oauth2-proxy-bin/):

   El paquete **oauth2-proxy-bin** descarga un binario para las arquitecturas soportadas un release de **oauth2-proxy-bin** en upstream.

   Ventajas:

   - Provee una versión considerada estable del paquete
   - No requiere compilar un binario

   Desventajas:

   - Puede no haber binarios en upstream para ciertas arquitecturas
   - Puede que el binario no esté optimizado para tu equipo
   - Puede que el binario no esté hardenizado

2. [oauth2-proxy-git](https://aur.archlinux.org/packages/oauth2-proxy-git/):

   El paquete **oauth2-proxy-bin** descarga y compila el código fuente del último commit de **oauth2-proxy-bin** en upstream.

   Ventajas:

   - Provee los cambios más recientes publicados en upstream
   - El **PKGBUILD** es fácil de mantener pues no se hace validación por checksums
   - El binario se puede compilar para múltiples arquitecturas
   - El binario se puede optimizar para tu equipo

   Desventajas:

   - Compilarlo puede ser tardado en ciertos equipos
   - Puede tener cambios considerados inestables

Aunque los 3 paquetes tienen un nombre diferente en el **AUR** el paquete que instalan debe llamarse **oauth2-proxy** y tener conflicto con **oauth2-proxy** para evitar que [pacman](https://wiki.archlinux.org/title/Pacman_(Espa%C3%B1ol)) instale diferentes versiones de un mismo paquete al mismo tiempo[^conflictos-mismo-paquete-aur]:

```bash
pkgname=oauth2-proxy
provides=($pkgname)
conflicts=($pkgname)
```

## Contribuyendo al AUR

Una de las primeras maneras de contribuir a Arch Linux y el primer paso para ser [Trusted User](https://wiki.archlinux.org/title/Trusted_Users_(Espa%C3%B1ol)) es mantener paquetes para el AUR[^trusted-user-mantener-paquetes].

Sigo los siguientes pasos al empaquetar software para el AUR.

0. [Escoger lo que voy a empaquetar](#escoger-lo-que-voy-a-empaquetar)
1. [Definir lo que un PKGBUILD debe instalar](#definir-lo-que-un-pkgbuild-debe-instalar)
2. [Validar el PKGBUILD](#validar-el-pkgbuild)
3. [Actualizar checksums](#actualizar-checksums)
4. [Construir el paquete](#construir-el-paquete)
5. [Instalar el paquete](#instalar-el-paquete)
6. [Generar .SRCINFO](#generar-srcinfo)
7. [Publicar el paquete](#publicar-el-paquete)

### Escoger lo que voy a empaquetar

El primer paso es verificar que lo que queremos empaquetar no se encuentre disponible ni en los [Repositorios Oficiales](https://wiki.archlinux.org/title/Official_repositories_(Espa%C3%B1ol)) ni en el **AUR** para evitar tener paquetes duplicados.[^paquetes-duplicados-en-aur]

Para crear el repositorio de un paquete nuevo con [git(1)](https://man.archlinux.org/man/git.1):

```bash
$ git clone ssh://aur@aur.archlinux.org/<PAQUETE>.git
```

Para clonar el repositorio de un [paquete huérfano](https://aur.archlinux.org/packages/?SB=n&do_Orphans=Orphans) con [aurpublish(1)](https://man.archlinux.org/man/aurpublish.1):

```bash
$ aurpublish -p <PAQUETE>
```

### Definir lo que un PKGBUILD debe instalar

Un **PKGBUILD** debe instalar lo siguiente:

0. Instalar el paquete.
1. Instalar la licencia el paquete usa una licencia que no se encuentra en el archivo [licenses](https://archlinux.org/packages/core/any/licenses/) (ej. [MIT](https://en.wikipedia.org/wiki/MIT_License)).[^instalar-licencia-pkgbuild]
2. Instalar la configuración por defecto si existe en upstream.
3. Instalar los [demonios](https://wiki.archlinux.org/title/Daemons_(Espa%C3%B1ol)) si existen en upstream.

Y debe hacerlo siguiendo la [Guía para paquetes de Arch](https://wiki.archlinux.org/title/Arch_package_guidelines_(Espa%C3%B1ol)), la guía para paquetes específica del lenguaje que estás empaquetando y las [Reglas de envío del AUR](https://wiki.archlinux.org/title/Arch_User_Repository_(Espa%C3%B1ol)#Reglas_de_env%C3%ADo).

El [PKGBUILD](https://github.com/da-edra/pkgbuilds/blob/trunk/oauth2-proxy/PKGBUILD) que escribí para **oauth2-proxy** seguí la Guía para paquetes de Arch y la [Guía para paquetes de Go](https://wiki.archlinux.org/title/Go_package_guidelines) para obtener un binario hardenizado usando compilación [PIE](https://en.wikipedia.org/wiki/Position-independent_code) y mejorar la [reproducibilidad del paquete](https://wiki.archlinux.org/title/Reproducible_Builds):

```shell
# Maintainer: Andrea Denisse Gómez-Martínez <denisse at archlinux dot org>

pkgname=oauth2-proxy
pkgdesc='A reverse proxy that provides authentication with Google, Github or other providers.'
arch=(any)
url='https://oauth2-proxy.github.io/oauth2-proxy/'
_url='https://github.com/oauth2-proxy/oauth2-proxy'
_branch='master'
pkgver=7.2.0
pkgrel=1
license=('MIT')
makedepends=(go sed)
source=("${pkgname}-${pkgver}.tar.gz::${_url}/archive/v${pkgver}.tar.gz")
b2sums=('cb40e2be2ab335289d2785382fedb3f57cd9b1f7d67311d247e692e22a135f51cf460b534196981e9e87f31ab44500f92d5e938701f2a45971a0f20178f66dd5')
provides=($pkgname)
conflicts=($pkgname)

prepare() {
  cd "${pkgname}-${pkgver}"
  mkdir -p build
  sed -i 's|/usr/local/bin/oauth2-proxy|/usr/bin/oauth2-proxy|' "contrib/$pkgname.service.example"
}

check() {
  cd "${pkgname}-${pkgver}"
  go test ./...
}

build() {
  cd "${pkgname}-${pkgver}"

  export CGO_LDFLAGS="${LDFLAGS}"
  export CGO_CFLAGS="${CFLAGS}"
  export CGO_CPPFLAGS="${CPPFLAGS}"
  export CGO_CXXFLAGS="${CXXFLAGS}"
  export GOFLAGS="-buildmode=pie -trimpath -ldflags=-linkmode=external -mod=readonly -modcacherw"

  go build -o build/ .
}

package() {
  cd "${pkgname}-${pkgver}"

  install -Dm755 "build/$pkgname" "$pkgdir/usr/bin/$pkgname"
  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
  install -Dm644 "contrib/$pkgname.cfg.example" "$pkgdir/etc/oauth2-proxy.cfg"
  install -Dm644 "contrib/$pkgname.service.example" "$pkgdir/usr/lib/systemd/system/oauth2-proxy.service"
}
```

### Validar el PKGBUILD

Para buscar errores comunes en el PKGBUILD uso [namcap(1)](https://man.archlinux.org/man/namcap.1):

```bash
$ namcap PKGBUILD
```

Algunos paquetes incluyen una función **check()** en la que se pueden agregar validaciones adicionales como ejecutar los tests que el paquete contiene upstream.[^función-check]

La configuración por defecto del archivo **makepkg.conf** ejecuta la función **check()** si ésta se encuentra en el **PKGBUILD**.[^config-makepkg.conf]. **check()** también se puede ejecutar manualmente:

```bash
$ makepkg --check PKGBUILD
```

### Actualizar checksums

El **PKGBUILD** de paquetes de tipo **PAQUETE** y **PAQUETE-bin** debe tener **checksums** que permiten verificar la integridad del paquete bajado de upstream.

Si un paquete incluye **checksums** en upstream es importante usar esos **checksums** en el **PKGBUILD** para garantizar la integridad del paquete desde su publicación en upstream hasta su instalación en local:[^checksums-integridad-de-paquete]

```bash
b2sums=('79586b8592a246ddb9a75752329e94bcc8d8924dcbe2eb2c7bdd1a11981e4ee39abcea86fb7b76e8c54dc8dd0f20d8b5d4b5f63025380f1ed9efbcca8c9b0bb7')
```

Si un paquete no cuenta con hashes en upstream estos se pueden generar con [updpkgsums(8)](https://man.archlinux.org/man/updpkgsums.8.en) :

```bash
updpkgsums PKGBUILD
```

En el caso de **PACKAGE-git** se debe saltar la validación de checksums:

```bash
b2sums=(SKIP)
```

### Construir el paquete

Los [paquetes que mantengo](https://aur.archlinux.org/packages/?O=0&SeB=M&K=denisse&outdated=&SB=n&SO=a&PP=50&do_Search=Go) están organizados en carpetas que llevan el nombre del paquete. Cada carpeta contiene un archivo **PKGBUILD** y un archivo [.SRCINFO](https://wiki.archlinux.org/title/.SRCINFO):

```bash
.
├── oaut2-proxy
│  ├── .SRCINFO
│  └── PKGBUILD
├── oaut2-proxy-bin
│  ├── .SRCINFO
│  └── PKGBUILD
└── oaut2-proxy-git
   ├── .SRCINFO
   └── PKGBUILD
```

Construyo el paquete con [makepkg(8)](https://man.archlinux.org/man/makepkg.8.en):

```bash
$ makepkg --clean --syncdeps --force
```

**--clean** limpia el directorio después de construir el paquete

**--syncdeps** instala dependencias faltantes

**--force** sobrescribe un paquete ya construido

### Instalar el paquete

Instalo el paquete con [makepkg(8)](https://man.archlinux.org/man/makepkg.8.en):

```bash
$ makepkg --install
```

Realizo las siguientes validaciones:

0. El paquete funciona como se espera.
1. Si se necesitan licencias éstas están instaladas.
2. Si hay una configuración por defecto ésta está instalada y funciona.
3. Si en upstream hay demonios estos están instalados y funcionan.

### Generar .SRCINFO

Genero el archivo **.SRCINFO** con [makepkg(8)](https://man.archlinux.org/man/makepkg.8.en). **.SRCINFO** contiene metadatos organizados en pares de `llave = valor` que son más fáciles y seguros de parsear que un PKGBUILD.[^problemas-parseando-pkgbuilds]

```bash
$ makepkg --printsrcinfo > .SRCINFO
```

### Publicar el paquete

El último paso es usar [aurpublish(1)](https://man.archlinux.org/man/aurpublish.1) para publicar el paquete:

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

Actualizando un **PKGBUILD**:

```bash
make update-pkg pkg=PACKAGE
```

Publicando un **PKGBUILD**:

```bash
make publish pkg=PACKAGE
```

[^conflictos-mismo-paquete-aur]: [Conflictos en paquetes del AUR](https://wiki.archlinux.org/title/PKGBUILD#conflicts)
[^trusted-user-mantener-paquetes]: [Requisitos mínimos para ser Trusted User](https://wiki.archlinux.org/title/Trusted_Users_(Espa%C3%B1ol)#%C2%BFC%C3%B3mo_me_convierto_en_un_TU?)
[^instalar-licencia-pkgbuild]: [Instalar la LICENCIA del paquete](https://wiki.archlinux.org/title/PKGBUILD#license)
[^config-makepkg.conf]: [Configuración por defecto de makepkg.conf](https://man.archlinux.org/man/makepkg.conf.5.en)
[^función-check]: [Función check() de un PKGBUILD](https://wiki.archlinux.org/title/Creating_packages#check())
[^paquetes-duplicados-en-aur]: [Reglas de envío para el AUR](https://wiki.archlinux.org/title/Arch_User_Repository_(Espa%C3%B1ol)#Reglas_de_env%C3%ADo)
[^checksums-integridad-de-paquete]:  [Validar la integridad del paquete con checksums](https://wiki.archlinux.org/title/PKGBUILD#Integrity)
[^problemas-parseando-pkgbuilds]: Diferentes problemas encontrados en al parsear PKGBUILD: [FS#25210](https://bugs.archlinux.org/task/25210), [FS#15043](https://bugs.archlinux.org/task/15043), y [FS#16394](https://bugs.archlinux.org/task/16394)
