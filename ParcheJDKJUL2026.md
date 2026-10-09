# Parche JDK JUL2026 – ExaCC 4 nodos

## Alcance

| Home | Método | Usuario |
|---|---|---|
| `/u01/app/19.0.0.0/grid` | `opatchauto` | root (con el entorno cargado) |
| `/u02/app/oracle/product/19.0.0.0/dbhome_1` | `opatch` | oracle |
| `/u02/app/oracle/product/19.0.0.0/dbhome_2` | `opatch` | oracle |

## Ficheros

Staging en ACFS compartido (visible en los 4 nodos): **`/acfs01/acfs/evolutivos`**

| Zip | Contenido |
|---|---|
| `JUL2026_p39329591_190000_Linux-x86-64.zip` | Parche JDK julio 2026 (Grid y RDBMS) |
| `OPATCH_1220152_p6880880_190000_Linux-x86-64.zip` | OPatch 12.2.0.1.52 (requisito del parche) |

- **Parche JDK:** se descomprime **una vez** en el ACFS (paso 0).
- **OPatch:** se descomprime **dentro de cada home**, en cada nodo, porque cada Oracle Home lleva su propia copia.

## Criterios

- **Nodo a nodo:** se termina un nodo completo (Grid + dbhome_1 + dbhome_2) antes de pasar al siguiente.
- **Si el analyze o un precheck falla, no se lanza el apply.**
- **Permisos:** `chmod` siempre en formato numérico.

## Secuencia de usuarios (desde `exaopc`)

| Orden | Usuario | Cómo se entra | Qué se hace |
|---|---|---|---|
| 1 | **grid** | `sudo su - grid` | Paso 0a: descomprimir el parche JDK en el ACFS (solo nodo 1) |
| 2 | **root** | `sudo su - root` | Paso 0b: owner de los zip (solo nodo 1) + Bloque A: Grid con `opatchauto` |
| 3 | **oracle** | `sudo su - oracle` | Bloques B y C: dbhome_1 y dbhome_2 con `opatch` |

En los nodos 2, 3 y 4 se empieza directamente en el paso 2 (root, bloque A).

---

## 0a. Descomprimir el parche JDK (como grid, solo nodo 1)

```bash
sudo su - grid
```

```bash
mv /acfs01/acfs/evolutivos/39329591 /acfs01/acfs/evolutivos/39329591_old_20261008
unzip /acfs01/acfs/evolutivos/JUL2026_p39329591_190000_Linux-x86-64.zip -d /acfs01/acfs/evolutivos
ls -ld /acfs01/acfs/evolutivos/39329591
exit
```

- El `mv` aparta el directorio `39329591` del 26 de junio, que ya existía, para descomprimir uno limpio desde el zip.
- Al descomprimir como grid, el directorio `39329591` ya queda con owner **grid**. Como oracle está en el grupo **oinstall**, puede leerlo.

## 0b. Owner de los zip (como root, solo nodo 1)

```bash
sudo su - root
```

```bash
chown grid:oinstall /acfs01/acfs/evolutivos/JUL2026_p39329591_190000_Linux-x86-64.zip
chown grid:oinstall /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip
chmod 755 /acfs01/acfs/evolutivos/JUL2026_p39329591_190000_Linux-x86-64.zip
chmod 755 /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip
ls -l /acfs01/acfs/evolutivos
```

Sin salir de root, se continúa con el bloque A.

En los nodos 2, 3 y 4, antes del bloque A, comprobar que se ve el parche:

```bash
ls -ld /acfs01/acfs/evolutivos/39329591
```

---

# A. GRID — `/u01/app/19.0.0.0/grid` (opatchauto, como root)

## A0. Verificar el puntero del inventario central (como root, cada nodo de Norte)

En Norte, `/etc/oraInst.loc` puede ser un enlace a `/etc/oraInst_oraemagent.loc`, que apunta a `/oraemagent/app/oraInventory`, un directorio que no existe. Con ese puntero, el `opatchauto` de A4 falla con `OiiiInventoryDoesNotExistException`. En Sur el enlace es correcto.

```bash
sudo su - root
ls -l /etc/oraInst.loc
cat /etc/oraInst.loc
```

Si apunta a `/u01/app/oraInventory/oraInst.loc` y el contenido es `inventory_loc=/u01/app/oraInventory` / `inst_group=oinstall`, el nodo está bien y se sigue en A1.

Si apunta a cualquier otro sitio, se rehace el enlace igual que en Sur. Por ejemplo, si apunta a `/etc/oraInst_oraemagent.loc`, o a `/u01/app/19.0.0.0/grid/oraInst.loc` (aunque el contenido sea correcto, para que quede idéntico a Sur):

```bash
cp /etc/oraInst.loc /etc/oraInst.loc_bk_20261009
cat /u01/app/oraInventory/oraInst.loc
rm /etc/oraInst.loc
ln -s /u01/app/oraInventory/oraInst.loc /etc/oraInst.loc
ls -l /etc/oraInst.loc
cat /etc/oraInst.loc
```

- El `cp` guarda el contenido original como backup.
- El segundo `cat` confirma que existe el fichero de destino del nuevo enlace. Si no existe, no lances el `rm`.
- El `rm` solo borra el enlace, no el fichero al que apunta.
- El resultado debe ser `/etc/oraInst.loc -> /u01/app/oraInventory/oraInst.loc`, con `inventory_loc=/u01/app/oraInventory` e `inst_group=oinstall`.

## A1. Cargar el entorno (como root)

Se sigue como root desde A0.

```bash
cd /home/grid
. /home/grid/.bash_profile
. /home/grid/.bashrc
```

## A2. Versión actual

```bash
/u01/app/19.0.0.0/grid/OPatch/opatch version
/u01/app/19.0.0.0/grid/OPatch/jre/bin/java -version
/u01/app/19.0.0.0/grid/jdk/bin/java -version
```

## A3. Actualizar OPatch

```bash
mv /u01/app/19.0.0.0/grid/OPatch /u01/app/19.0.0.0/grid/OPatch_old_20261008
unzip /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip -d /u01/app/19.0.0.0/grid
chown -R grid:oinstall /u01/app/19.0.0.0/grid/OPatch
chmod 755 /u01/app/19.0.0.0/grid/OPatch
ls -ld /u01/app/19.0.0.0/grid/OPatch
/u01/app/19.0.0.0/grid/OPatch/opatch version
/u01/app/19.0.0.0/grid/OPatch/jre/bin/java -version
```

`opatch version` debe dar **12.2.0.1.52**.

## A4. Analyze

```bash
cd /acfs01/acfs/evolutivos
/u01/app/19.0.0.0/grid/OPatch/opatchauto apply /acfs01/acfs/evolutivos/39329591 -oh /u01/app/19.0.0.0/grid -analyze
```

## A5. Apply

```bash
/u01/app/19.0.0.0/grid/OPatch/opatchauto apply /acfs01/acfs/evolutivos/39329591 -oh /u01/app/19.0.0.0/grid
```

## A6. Comprobar

```bash
/u01/app/19.0.0.0/grid/jdk/bin/java -version
/u01/app/19.0.0.0/grid/jdk/jre/bin/java -version
/u01/app/19.0.0.0/grid/OPatch/jre/bin/java -version
/u01/app/19.0.0.0/grid/bin/crsctl check crs
```

El `lspatches` se lanza como grid; después, `exit` para volver a root y continuar:

```bash
su - grid
/u01/app/19.0.0.0/grid/OPatch/opatch lspatches -oh /u01/app/19.0.0.0/grid
exit
```

---

# B. RDBMS — dbhome_1 (opatch, como oracle)

## B1. Versión actual

```bash
sudo su - oracle
```

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/opatch version
/u02/app/oracle/product/19.0.0.0/dbhome_1/jdk/bin/java -version
```

## B2. Actualizar OPatch

```bash
mv /u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch /u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch_old_20261008
unzip /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip -d /u02/app/oracle/product/19.0.0.0/dbhome_1
ls -ld /u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch
/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/opatch version
```

## B3. Prechecks

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/opatch prereq CheckActiveFilesAndExecutables -ph /acfs01/acfs/evolutivos/39329591 -oh /u02/app/oracle/product/19.0.0.0/dbhome_1
/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph /acfs01/acfs/evolutivos/39329591 -oh /u02/app/oracle/product/19.0.0.0/dbhome_1
```

## B4. Apply

```bash
cd /acfs01/acfs/evolutivos/39329591
/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/opatch apply -oh /u02/app/oracle/product/19.0.0.0/dbhome_1 -local -silent
```

## B5. Comprobar

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_1/jdk/bin/java -version
/u02/app/oracle/product/19.0.0.0/dbhome_1/jdk/jre/bin/java -version
/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/opatch lspatches -oh /u02/app/oracle/product/19.0.0.0/dbhome_1
```

---

# C. RDBMS — dbhome_2 (opatch, como oracle)

## C1. Versión actual

Se sigue como oracle desde el bloque B.

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/opatch version
/u02/app/oracle/product/19.0.0.0/dbhome_2/jdk/bin/java -version
```

## C2. Actualizar OPatch

```bash
mv /u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch /u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch_old_20261008
unzip /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip -d /u02/app/oracle/product/19.0.0.0/dbhome_2
ls -ld /u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch
/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/opatch version
```

## C3. Prechecks

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/opatch prereq CheckActiveFilesAndExecutables -ph /acfs01/acfs/evolutivos/39329591 -oh /u02/app/oracle/product/19.0.0.0/dbhome_2
/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph /acfs01/acfs/evolutivos/39329591 -oh /u02/app/oracle/product/19.0.0.0/dbhome_2
```

## C4. Apply

```bash
cd /acfs01/acfs/evolutivos/39329591
/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/opatch apply -oh /u02/app/oracle/product/19.0.0.0/dbhome_2 -local -silent
```

## C5. Comprobar

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_2/jdk/bin/java -version
/u02/app/oracle/product/19.0.0.0/dbhome_2/jdk/jre/bin/java -version
/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/opatch lspatches -oh /u02/app/oracle/product/19.0.0.0/dbhome_2
```

---

# D. Limpieza: zipear el OPatch antiguo (en cada nodo)

Cuando el nodo esté validado, cada `OPatch_old_20261008` se convierte en un zip dentro de su propio home. Así el escáner deja de detectar el jre antiguo.

Hace falta `-f` en el `rm`: con `rm -r` solo, pide confirmación por cada fichero protegido contra escritura.

## D1. Grid (como root)

```bash
sudo su - root
cd /u01/app/19.0.0.0/grid
zip -r OPatch_old_20261008.zip OPatch_old_20261008
rm -rf OPatch_old_20261008
chown grid:oinstall OPatch_old_20261008.zip
chmod 640 OPatch_old_20261008.zip
ls -l OPatch_old_20261008*
```

## D2. dbhome_1 (como oracle)

```bash
sudo su - oracle
cd /u02/app/oracle/product/19.0.0.0/dbhome_1
zip -r OPatch_old_20261008.zip OPatch_old_20261008
rm -rf OPatch_old_20261008
ls -l OPatch_old_20261008*
```

## D3. dbhome_2 (como oracle)

```bash
cd /u02/app/oracle/product/19.0.0.0/dbhome_2
zip -r OPatch_old_20261008.zip OPatch_old_20261008
rm -rf OPatch_old_20261008
ls -l OPatch_old_20261008*
```

En cada home, el `ls -l` final solo debe mostrar el `.zip`.

---

# E. Limpieza: zipear el JDK del oraemagent (en cada nodo, como root)

```bash
sudo su - root
cd /oraemagent/app/oracle/middleware/agent_13.5.0.0.0/oracle_common
zip -r jdk_old_20261008.zip jdk
rm -rf jdk
```

---

## Orden de ejecución

1. **Solo en el nodo 1, una vez:** 0a (grid) → 0b (root). El parche queda descomprimido en el ACFS para los 4 nodos.
2. **En cada nodo:**
   1. **A (root):** en Norte, verificar `/etc/oraInst.loc` (A0); cargar el entorno, actualizar OPatch, `opatchauto -analyze` y `opatchauto apply`. En A6, `su - grid` para el `lspatches` y `exit` para volver a root.
   2. **B (oracle):** dbhome_1 con `opatch`.
   3. **C (oracle):** dbhome_2 con `opatch`, sin cambiar de usuario.
3. Comprobar que el CRS, las instancias y los servicios están bien antes de pasar al siguiente nodo.
4. **Limpieza en cada nodo, una vez validado:**
   1. **D1 (root):** zip del `OPatch_old_20261008` del Grid.
   2. **D2 y D3 (oracle):** zip del `OPatch_old_20261008` de dbhome_1 y dbhome_2.
   3. **E (root):** zip del `jdk` del oraemagent.

## Notas

- **Rutas:** absolutas en todo el documento salvo en los apartados D y E, que usan rutas relativas tras el `cd` a cada directorio.
- **Staging:** `/acfs01/acfs/evolutivos` es un ACFS compartido y visible en los 4 nodos. El parche JDK se descomprime una sola vez, como grid. Los dos zip tienen owner `grid:oinstall`.
- **OPatch se descomprime desde el zip en cada home y nodo.** No se mueve un directorio desde el ACFS, porque tras el primer nodo ya no existiría para el resto.
- **`chown -R` en el Grid**, para que todo el contenido de `OPatch` quede como grid:oinstall.
- **`opatchauto` actúa solo en el nodo local.** No aplica el parche en los 4 nodos a la vez.
- **`zip` y `unzip` sin `-q`**, para ver en pantalla la lista de ficheros.
- **`rm -rf`:** el `-f` es necesario; con `rm -r` solo, pide confirmación por cada fichero protegido contra escritura.
- **Oraemagent:** sin su `jdk`, el agente de EM no puede arrancar. Para restaurarlo: `unzip /oraemagent/app/oracle/middleware/agent_13.5.0.0.0/oracle_common/jdk_old_20261008.zip -d /oraemagent/app/oracle/middleware/agent_13.5.0.0.0/oracle_common`.
- **Inventario central en Norte:** `/etc/oraInst.loc` puede apuntar al inventario del agente de EM, que no existe, y eso rompe `opatchauto`. Se corrige en A0 rehaciendo el enlace a `/u01/app/oraInventory/oraInst.loc`, como en Sur.
- **`.patch_storage`** conserva el JDK anterior como backup. Si el escáner de vulnerabilidades lo vuelve a marcar, el origen es ese.

### Rutas del informe y dónde se resuelven

| Ruta | Versión vulnerable | Sección |
|---|---|---|
| `/u01/app/19.0.0.0/grid/OPatch/jre/bin/java` | 1.8.0_441-b07 | A3 + D1 |
| `/u01/app/19.0.0.0/grid/jdk/bin/java` | 1.8.0_441-b07 | A5 |
| `/u01/app/19.0.0.0/grid/jdk/jre/bin/java` | 1.8.0_441-b07 | A5 |
| `/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/jre/bin/java` | 1.8.0_451-b09 | B2 + D2 |
| `/u02/app/oracle/product/19.0.0.0/dbhome_1/jdk/bin/java` | 1.8.0_441-b07 | B4 |
| `/u02/app/oracle/product/19.0.0.0/dbhome_1/jdk/jre/bin/java` | 1.8.0_441-b07 | B4 |
| `/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/jre/bin/java` | 1.8.0_451-b09 | C2 + D3 |
| `/u02/app/oracle/product/19.0.0.0/dbhome_2/jdk/bin/java` | 1.8.0_441-b07 | C4 |
| `/u02/app/oracle/product/19.0.0.0/dbhome_2/jdk/jre/bin/java` | 1.8.0_441-b07 | C4 |
| `/oraemagent/app/oracle/middleware/agent_13.5.0.0.0/oracle_common/jdk/bin/java` | 1.8.0_261-b12 | E |
| `/oraemagent/app/oracle/middleware/agent_13.5.0.0.0/oracle_common/jdk/jre/bin/java` | 1.8.0_261-b12 | E |
