# Histórico

## `index-original.html`

La aplicación ENTERA antes de la migración a Vue 3, en un solo archivo de 8.591
líneas. Venía de `github.com/maldoxs/maldoxfilm`, el repositorio viejo.

**Por qué se conserva.** El código actual tiene **450 comentarios en 55
archivos** que citan líneas de este archivo, del estilo "preservado de las
líneas ~7798-7813". Sin él, esas referencias no se pueden seguir y se pierde la
trazabilidad de la migración: qué hacía el original en cada punto y por qué la
versión nueva hace lo que hace.

No es código vivo. No se ejecuta, no se mantiene, no se corrige. Es la fuente
que esos comentarios citan.

**Una modificación respecto del original:** la línea `var RD_TOKEN = '...'`
tenía el token de Real-Debrid escrito completo. Se reemplazó por un texto. Ese
token quedó comprometido por estar publicado y no debe reusarse (ver
`architecture/security.md`).
