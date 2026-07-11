# Candyworks — Simulador de Trocas

Site estático em HTML, CSS e JavaScript para simular as trocas mostradas na imagem.

## Como abrir

Abra o arquivo `index.html` diretamente no navegador.

Também é possível iniciar um servidor local:

```bash
python -m http.server 8000
```

Depois acesse `http://localhost:8000`.

## Recursos

- Estoque inicial configurado com os doces da imagem.
- Meta da Arcana da Windranger.
- Quatro receitas configuráveis pelo código.
- Troca personalizada de 3 doces iguais por 1 escolhido.
- Histórico e desfazer.
- Salvar/carregar no `localStorage`.
- Verificação matemática por pontuação.
- Busca automática por largura (BFS) com limite de segurança.
- Layout responsivo para celular e computador.

## Observação

O simulador assume que a troca personalizada é “3 doces iguais por 1 doce escolhido”.
