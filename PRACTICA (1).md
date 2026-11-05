# Práctica: conflictos, `--no-ff`, `--squash` y `rebase`

Punto de partida: `main` limpio, la última sección de `index.html` es la 3.1 y todavía no hay README.
Haz las 4 partes **en orden** y ejecuta `git log --oneline --graph` al final de cada una para tener la evidencia.

> Consejo: crea un alias para el comando de evidencia:
> ```bash
> git config --global alias.g "log --oneline --graph --all"
> ```
> A partir de aquí, `git g` muestra el grafo de todas las ramas.

---

## A) Fusión de tres vías con conflicto (sección 3.2)

**Concepto:** cuando `main` y la rama avanzan cada una por su lado, Git compara tres versiones: el ancestro común, la punta de `main` y la punta de la rama. Si las dos cambiaron **la misma línea**, Git no sabe con cuál quedarse y marca un conflicto.

```bash
git switch -c feature/seccion-3-2
```

**Cambio en `index.html` (en la rama):** modifica el `<p>` de la 3.1 y añade la sección 3.2 justo debajo:

```html
        <section>
            <h2>3.1 Ramas</h2>
            <p>Una rama es una línea de trabajo independiente que apunta a un commit.</p>
            <code>git switch -c feature/nueva-idea</code>
        </section>
        <section>
            <h2>3.2 Fusiones y conflictos</h2>
            <p>Un conflicto aparece cuando dos ramas cambian la misma línea.</p>
            <code>git merge feature/seccion-3-2</code>
        </section>
```

```bash
git add index.html
git commit -m "Agrega sección 3.2 y redefine rama"
git switch main
```

**Cambio en `index.html` (en main):** cambia **la misma línea** `<p>` de la 3.1, pero con otro texto:

```html
            <p>Una rama me permite experimentar sin romper main.</p>
```

```bash
git add index.html
git commit -m "Cambia definición de rama en main"
git g        # verás dos líneas que divergen
git merge feature/seccion-3-2
# CONFLICT (content): Merge conflict in index.html
git status   # index.html aparece como "both modified"
```

Al abrir `index.html` verás los marcadores:

```
<<<<<<< HEAD
            <p>Una rama me permite experimentar sin romper main.</p>
=======
            <p>Una rama es una línea de trabajo independiente que apunta a un commit.</p>
>>>>>>> feature/seccion-3-2
```

- `HEAD` es lo que tiene `main`, la rama donde estás.
- Lo de abajo es lo que viene de la rama.

**Para resolverlo**, deja una sola versión (o combina las dos) y **borra las tres líneas de marcadores**:

```html
            <p>Una rama es una línea de trabajo independiente: experimento sin romper main.</p>
```

```bash
git add index.html          # marca el conflicto como resuelto
git commit                  # Git propone "Merge branch 'feature/seccion-3-2'"; guarda y sal
git g
```

**Evidencia esperada:** dos líneas que se separan y se unen en un commit de merge con dos padres.

```
*   a1b2c3d Merge branch 'feature/seccion-3-2'
|\
| * ...    Agrega sección 3.2 y redefine rama
* | ...    Cambia definición de rama en main
|/
* 931dd08 fix/formato
```

> Si te equivocas a mitad, `git merge --abort` deja todo como estaba antes del merge.

---

## B) `--no-ff` (nuevo pie de página)

**Concepto:** si `main` no avanzó, Git haría un *fast-forward*: movería el puntero de `main` sin crear ningún commit, y la rama "desaparecería" del historial. Con `--no-ff` obligas a crear un commit de merge, de modo que queda constancia de que hubo una rama.

```bash
git switch -c feature/footer
```

**Cambio en `index.html`:**

```html
    <footer>
        <p>Hecho por Cristopher Larios y Roderick David Zapata con Git</p>
        <p>Curso de Git y GitHub · 2026</p>
    </footer>
```

```bash
git add index.html
git commit -m "Nuevo pie de página"
git switch main
git merge --no-ff feature/footer -m "Merge feature/footer (--no-ff)"
git g
```

**Evidencia esperada:** aparece una "burbuja" aunque `main` no se haya movido.

```
*   ... Merge feature/footer (--no-ff)
|\
| * ... Nuevo pie de página
|/
*   ... Merge branch 'feature/seccion-3-2'
```

> Para explicar la diferencia: sin `--no-ff` no habría burbuja, solo `* Nuevo pie de página` en línea recta.

---

## C) `--squash` (tres retoques de CSS en un solo commit)

**Concepto:** junta todos los cambios de la rama en **un único commit nuevo** en `main`. No hay commit de merge ni enlace con la rama, así que el grafo queda lineal.

```bash
git switch -c feature/css
```

Haz **tres commits separados** en `css/main.css`:

1. Cambia el color de los títulos:
   ```css
   h2 {
       color: #1d4ed8;
       margin-top: 0;
   }
   ```
   ```bash
   git commit -am "CSS: color de títulos"
   ```
2. Añade dentro de `section { ... }`:
   ```css
       border-left: 6px solid #f9a27e;
   ```
   ```bash
   git commit -am "CSS: borde lateral en secciones"
   ```
3. Añade dentro de `code { ... }`:
   ```css
       font-size: 14px;
   ```
   ```bash
   git commit -am "CSS: tamaño de código"
   ```

```bash
git g                         # la rama tiene 3 commits
git switch main
git merge --squash feature/css
git status                    # los cambios están en staged, todavía SIN commit
git commit -m "Retoques de CSS (squash de 3 commits)"
git g
```

**Evidencia esperada:** en `main` hay un solo commit lineal, y los 3 commits originales siguen colgando en la rama `feature/css` sin unirse.

```
* ... Retoques de CSS (squash de 3 commits)
| * ... CSS: tamaño de código
| * ... CSS: borde lateral en secciones
| * ... CSS: color de títulos
|/
*   ... Merge feature/footer (--no-ff)
```

> Detalle útil para la explicación: `git branch -d feature/css` **falla**, porque Git no considera esa rama fusionada. Hay que borrarla con `-D`.

---

## D) `rebase` (nota en el README sobre un main que avanzó)

**Concepto:** `rebase` vuelve a aplicar tus commits **encima** de la punta actual de `main`. Esos commits se reescriben y reciben hashes nuevos, y el historial queda en línea recta, como si hubieras empezado la rama hoy.

```bash
git switch -c docs/readme
echo "# Proyecto-CI" > README.md
echo "Nota: bitácora del curso de Git; cada sección documenta una técnica." >> README.md
git add README.md
git commit -m "Agrega nota en README"
git switch main
```

**Ahora `main` avanza.** Cambio en `index.html`, en el `<header>`:

```html
        <p>Mi bitácora del curso de Git y GitHub · Unidad 3</p>
```

```bash
git commit -am "Actualiza subtítulo del header"
git g                         # docs/readme parte de un main viejo (divergen)
git switch docs/readme
git log --oneline -1          # anota el hash del commit
git rebase main
git log --oneline -1          # hash DISTINTO: el commit fue reescrito
git g                         # docs/readme ahora está encima de main
git switch main
git merge docs/readme         # fast-forward: sin commit de merge
git g
```

**Evidencia esperada:** línea recta, sin burbuja.

```
* ... Agrega nota en README
* ... Actualiza subtítulo del header
* ... Retoques de CSS (squash de 3 commits)
```

> Regla de oro: nunca hagas rebase de commits que ya subiste y comparte otra gente, porque les reescribes el historial.

---

## Resumen para explicar

| Técnica | ¿Crea un commit de merge? | ¿Conserva los commits de la rama? | Forma en `git log --graph` |
|---|---|---|---|
| Tres vías | Sí (con 2 padres) | Sí | Divergen y se unen |
| `--no-ff` | Sí, aunque se pudiera hacer FF | Sí | Burbuja |
| `--squash` | No | No: queda 1 commit nuevo | Línea recta; la rama queda suelta |
| `rebase` | No | Sí, pero reescritos (hash nuevo) | Línea recta |

## Limpieza final (opcional)

```bash
git branch -d feature/seccion-3-2 feature/footer docs/readme
git branch -D feature/css
git push origin main
```
