# Guia para git

## 1. Crear el repositorio en la cuenta `electronicapyd`
1. Inicia sesión en GitHub con la cuenta `electronicapyd`.
2. Crea un nuevo repositorio (ejemplo: `proyecto-en-el-que-estemosXD`).
3. No es necesario subir nada, puede quedar vacío o con un README inicial.

---

## 2. Dar permisos a colaboradores
1. En el repositorio recién creado, ve a **Settings → Collaborators → Manage access**.
2. Agrega las cuentas de GitHub de **Pablo** y **Daniel** como *collaborators* con permiso **Write**.
3. Así podrán hacer `push` y `pull` sin problemas.

---

## 3. Quien tenga el codigo o archivos sube su código local
En la PC del usuario local, donde ya tiene el proyecto:


## Inicializar git si aún no está
```bash
git init
```
## Conectar al repo remoto
```bash
git remote add origin https://github.com/electronicapyd/proyecto-principal.git
```

## Añadir todos los archivos
```bash
git add .
```

## Crear el primer commit
```bash
git commit -m "Primer commit del proyecto"
```

## Resolver error de master vs main
## Si tu rama local es 'master', renómbrala a 'main':
```bash
git branch -M main
```

## Subir al repositorio maestro
```bash
git push -u origin main
```

## Clonar el repo desde la cuenta maestro
```bash
git clone https://github.com/electronicapyd/proyecto-principal.git
cd proyecto-principal
```
---
# AVANZADO PERO RECOMENDABLE
---

## Crear una nueva rama para trabajar
```bash
git checkout -b feature-nueva-funcionalidad
```

## Editar archivos, luego añadir y hacer commit
```bash
git add .
git commit -m "Agrego nueva funcionalidad"
```

## Subir la rama al repositorio remoto
```bash
git push origin feature-nueva-funcionalidad
```

## Finalmente revisar las Pull requests 

## Revisar y hacer Merge

# Actualizar la rama local después del Merge

```bash
git checkout main
```

# Hacer pull de nuevos cambios xd

```bash
git pull origin main
```



