# Lectura 4 — Registros e imágenes (Docker Hub)
   - Qué es un registro, cómo se distribuyen imágenes.
   - Tags y versiones (concepto, sin entrar aún a Dockerfile).

## Preguntas de autoevaluación

1. Considerando que el rendimiento y la rapidez de los contenedores provienen de ejecutarse como procesos aislados sobre un kernel compartido, ¿qué decisiones de diseño (builder, runtime, rootless, seccomp/AppArmor/SELinux, red, filesystem por capas) tomaría para equilibrar **portabilidad**, **seguridad** y **desempeño** en un despliegue real, y cuáles serían sus criterios para justificar esos trade-offs?

## Referencias

- Tanenbaum, A., & Bos, H. *Modern Operating Systems* (4th ed.). Pearson. (Capítulo: Virtualization).
- Tankersley, C. *Docker for Developers*. Leanpub Publishing.
- Raj, P., & Sing, V. *Learning Docker*. Packt Publishing.
