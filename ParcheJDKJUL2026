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
- **Usuarios:** desde `exaopc` se entra con `sudo su - root` o `sudo su - oracle` según el bloque.

---

## 0. Preparar el staging (una sola vez, desde el nodo 1)

Los zip ya están descargados en `/acfs01/acfs/evolutivos`.

Como **oracle**:

```bash
unzip -qo /acfs01/acfs/evolutivos/JUL2026_p39329591_190000_Linux-x86-64.zip -d /acfs01/acfs/evolutivos
chmod -R o+rx /acfs01/acfs/evolutivos/39329591
chmod o+r /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip
```

Comprobar que los 4 nodos ven el parche:

```bash
dcli -l oracle -g ~/dbs_group "ls -ld /acfs01/acfs/evolutivos/39329591"
```

---

# A. GRID — `/u01/app/19.0.0.0/grid` (opatchauto, como root)

## A1. Cargar el entorno (como root)

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
unzip -q /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip -d /u01/app/19.0.0.0/grid
chown -R grid:oinstall /u01/app/19.0.0.0/grid/OPatch
chmod 755 /u01/app/19.0.0.0/grid/OPatch
ls -ld /u01/app/19.0.0.0/grid/OPatch
/u01/app/19.0.0.0/grid/OPatch/opatch version
/u01/app/19.0.0.0/grid/OPatch/jre/bin/java -version
```

`opatch version` debe dar **12.2.0.1.52**.

## A4. Analyze

```bash
export PATH=$PATH:/u01/app/19.0.0.0/grid/OPatch
cd /acfs01/acfs/evolutivos
opatchauto apply /acfs01/acfs/evolutivos/39329591 -oh /u01/app/19.0.0.0/grid -analyze
```

## A5. Apply

```bash
opatchauto apply /acfs01/acfs/evolutivos/39329591 -oh /u01/app/19.0.0.0/grid
```

## A6. Comprobar

```bash
/u01/app/19.0.0.0/grid/jdk/bin/java -version
/u01/app/19.0.0.0/grid/jdk/jre/bin/java -version
/u01/app/19.0.0.0/grid/OPatch/jre/bin/java -version
/u01/app/19.0.0.0/grid/OPatch/opatch lspatches -oh /u01/app/19.0.0.0/grid
/u01/app/19.0.0.0/grid/bin/crsctl check crs
```

---

# B. RDBMS — dbhome_1 (opatch, como oracle)

## B1. Versión actual

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch/opatch version
/u02/app/oracle/product/19.0.0.0/dbhome_1/jdk/bin/java -version
```

## B2. Actualizar OPatch

```bash
mv /u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch /u02/app/oracle/product/19.0.0.0/dbhome_1/OPatch_old_20261008
unzip -q /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip -d /u02/app/oracle/product/19.0.0.0/dbhome_1
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

```bash
/u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch/opatch version
/u02/app/oracle/product/19.0.0.0/dbhome_2/jdk/bin/java -version
```

## C2. Actualizar OPatch

```bash
mv /u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch /u02/app/oracle/product/19.0.0.0/dbhome_2/OPatch_old_20261008
unzip -q /acfs01/acfs/evolutivos/OPATCH_1220152_p6880880_190000_Linux-x86-64.zip -d /u02/app/oracle/product/19.0.0.0/dbhome_2
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

## Orden de ejecución

1. Paso 0 una sola vez, desde el nodo 1.
2. **Nodo 1:** A (root) → B (oracle) → C (oracle).
3. Comprobar que el CRS, las instancias y los servicios están bien en el nodo 1.
4. Repetir A → B → C en los **nodos 2, 3 y 4**.
5. Verificación final desde el nodo 1:

```bash
dcli -l oracle -g ~/dbs_group "/u01/app/19.0.0.0/grid/jdk/bin/java -version 2>&1 | head -1"
dcli -l oracle -g ~/dbs_group "/u02/app/oracle/product/19.0.0.0/dbhome_1/jdk/bin/java -version 2>&1 | head -1"
dcli -l oracle -g ~/dbs_group "/u02/app/oracle/product/19.0.0.0/dbhome_2/jdk/bin/java -version 2>&1 | head -1"
```

## Notas

- **OPatch se descomprime desde el zip en cada home y nodo.** No se mueve un directorio desde el ACFS, porque tras el primer nodo ya no existiría para el resto.
- **`chown -R` en el Grid**, para que todo el contenido de `OPatch` quede como grid:oinstall.
- **`.patch_storage`** conserva el JDK anterior como backup. Si el escáner de vulnerabilidades lo vuelve a marcar, el origen es ese.
