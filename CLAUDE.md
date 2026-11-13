# CLAUDE.md — market-recorder

## Contexto
- Recorder 24/7 de feeds WebSocket de exchanges, compartido por dos TFG de Miguel Prieto (UCM): el de Informática (Polymarket, Kalshi, spot) y el de Matemáticas (L3 de Kraken o Coinbase).
- Este repo solo graba raw, lo vuelca fuera del servidor y lo lee. La normalización a Parquet vive en cada proyecto consumidor.
- El contrato con los consumidores es el formato raw y `marketrec.raw.iter_raw_records`. Un cambio incompatible exige tag nuevo y avisar.
- Repositorio público y pieza de portfolio: prioriza claridad, tests y reproducibilidad.
- Diseño en `docs/design.md`; datos de APIs verificados en `docs/verification_log.md`.

## Cómo quiero trabajar contigo
- Respóndeme en español. Código, comentarios y README, en inglés.
- Quiero entender todo el código: pasos pequeños y revisables.
- Antes de implementar algo no trivial, propón el diseño y espera mi visto bueno.
- Cada módulo nuevo lleva tests (pytest).
- Si un dato de una API no está verificado, dilo y compruébalo en la documentación oficial antes de codificarlo.
- Nada de credenciales en el repo: variables de entorno o `.env` (en `.gitignore`).
- El recorder desplegado no se interrumpe sin avisar: los libros no se pueden recuperar después.

## Convenciones
- Python 3.12, uv, ruff, pyright, type hints.
- Timestamps en UTC, enteros en ns o ms, documentados por campo.
- Un contenedor por (exchange, ámbito), aislados entre sí.
