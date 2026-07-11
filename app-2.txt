const CANDIES = [
  { id: "green", name: "Verde", weight: 1 },
  { id: "white", name: "Branco", weight: 1 },
  { id: "blue", name: "Azul", weight: 1 },
  { id: "purple", name: "Roxo", weight: 3 },
  { id: "black", name: "Preto", weight: 3 }
];

const INITIAL = {
  green: 8,
  white: 10,
  blue: 3,
  purple: 1,
  black: 3
};

const TARGET = {
  green: 5,
  white: 2,
  blue: 0,
  purple: 4,
  black: 5
};

const RECIPES = [
  {
    id: "recipe-1",
    name: "Receita 1",
    input: { purple: 1, blue: 1 },
    output: { green: 2, white: 2 }
  },
  {
    id: "recipe-2",
    name: "Receita 2",
    input: { blue: 1, black: 2 },
    output: { green: 1, white: 3 }
  },
  {
    id: "recipe-3",
    name: "Receita 3",
    input: { green: 2, purple: 1 },
    output: { white: 2, black: 1 }
  },
  {
    id: "recipe-4",
    name: "Receita 4",
    input: { white: 1, blue: 1, black: 1 },
    output: { green: 2, purple: 1 }
  }
];

let inventory = { ...INITIAL };
let history = [];
let snapshots = [];
let toastTimer = null;

const inventoryGrid = document.querySelector("#inventoryGrid");
const targetGrid = document.querySelector("#targetGrid");
const recipesList = document.querySelector("#recipesList");
const historyList = document.querySelector("#historyList");
const solutionContent = document.querySelector("#solutionContent");
const customFrom = document.querySelector("#customFrom");
const customTo = document.querySelector("#customTo");
const customTradeBtn = document.querySelector("#customTradeBtn");
const targetTradeBtn = document.querySelector("#targetTradeBtn");
const undoBtn = document.querySelector("#undoBtn");
const currentScoreEl = document.querySelector("#currentScore");
const targetScoreEl = document.querySelector("#targetScore");
const totalCandiesEl = document.querySelector("#totalCandies");
const feasibilityBox = document.querySelector("#feasibilityBox");
const feasibilityTitle = document.querySelector("#feasibilityTitle");
const feasibilityText = document.querySelector("#feasibilityText");
const addCandyDialog = document.querySelector("#addCandyDialog");
const addCandyOptions = document.querySelector("#addCandyOptions");
const toast = document.querySelector("#toast");

function candyIcon(id) {
  return `<i class="candy-icon candy-${id}" aria-hidden="true"></i>`;
}

function getCandy(id) {
  return CANDIES.find(candy => candy.id === id);
}

function calculateScore(state) {
  return CANDIES.reduce(
    (sum, candy) => sum + (state[candy.id] || 0) * candy.weight,
    0
  );
}

function totalCandies(state) {
  return CANDIES.reduce((sum, candy) => sum + (state[candy.id] || 0), 0);
}

function hasTarget(state) {
  return CANDIES.every(candy => state[candy.id] >= TARGET[candy.id]);
}

function canApply(state, input) {
  return Object.entries(input).every(([id, amount]) => state[id] >= amount);
}

function applyDelta(state, input, output) {
  const next = { ...state };

  Object.entries(input).forEach(([id, amount]) => {
    next[id] -= amount;
  });

  Object.entries(output).forEach(([id, amount]) => {
    next[id] += amount;
  });

  return next;
}

function humanSide(side) {
  return Object.entries(side)
    .filter(([, amount]) => amount > 0)
    .map(([id, amount]) => `${amount} ${getCandy(id).name.toLowerCase()}`)
    .join(" + ");
}

function renderRecipeSide(side) {
  return Object.entries(side)
    .filter(([, amount]) => amount > 0)
    .map(([id, amount]) => `
      <span class="recipe-piece" title="${amount} ${getCandy(id).name}">
        ${candyIcon(id)}
        <b>${amount}</b>
      </span>
    `)
    .join("");
}

function renderInventory() {
  inventoryGrid.innerHTML = CANDIES.map(candy => `
    <article class="inventory-slot" title="${candy.name}">
      ${candyIcon(candy.id)}
      <strong>${inventory[candy.id]}</strong>
      <small>${candy.name.toUpperCase()}</small>
      <div class="slot-controls">
        <button
          type="button"
          data-change="-1"
          data-candy="${candy.id}"
          aria-label="Diminuir ${candy.name}"
        >−</button>
        <button
          type="button"
          data-change="1"
          data-candy="${candy.id}"
          aria-label="Aumentar ${candy.name}"
        >+</button>
      </div>
    </article>
  `).join("");

  inventoryGrid.querySelectorAll("[data-change]").forEach(button => {
    button.addEventListener("click", () => {
      const id = button.dataset.candy;
      const delta = Number(button.dataset.change);
      setInventoryManually(id, inventory[id] + delta);
    });
  });
}

function renderTarget() {
  targetGrid.innerHTML = CANDIES
    .filter(candy => TARGET[candy.id] > 0)
    .map(candy => `
      <span class="cost-piece" title="${TARGET[candy.id]} ${candy.name}">
        ${candyIcon(candy.id)}
        <b>${TARGET[candy.id]}</b>
      </span>
    `)
    .join("");
}

function renderRecipes() {
  recipesList.innerHTML = RECIPES.map(recipe => `
    <article class="recipe-row">
      <div class="recipe-side">${renderRecipeSide(recipe.input)}</div>
      <button
        class="recipe-go"
        data-recipe="${recipe.id}"
        title="${recipe.name}: ${humanSide(recipe.input)} para ${humanSide(recipe.output)}"
        aria-label="Aplicar ${recipe.name}"
      >▶</button>
      <div class="recipe-side">${renderRecipeSide(recipe.output)}</div>
    </article>
  `).join("");

  recipesList.querySelectorAll("[data-recipe]").forEach(button => {
    button.addEventListener("click", () => {
      const recipe = RECIPES.find(item => item.id === button.dataset.recipe);
      executeTrade(
        recipe.input,
        recipe.output,
        `${recipe.name}: ${humanSide(recipe.input)} → ${humanSide(recipe.output)}`
      );
    });
  });

  updateButtons();
}

function renderCustomOptions() {
  const options = CANDIES
    .map(candy => `<option value="${candy.id}">${candy.name}</option>`)
    .join("");

  customFrom.innerHTML = options;
  customTo.innerHTML = options;
  customFrom.value = "green";
  customTo.value = "purple";
}

function renderAddCandyOptions() {
  addCandyOptions.innerHTML = CANDIES.map(candy => `
    <button type="button" class="add-candy-option" data-add-candy="${candy.id}">
      ${candyIcon(candy.id)}
      <strong>${candy.name}</strong>
    </button>
  `).join("");

  addCandyOptions.querySelectorAll("[data-add-candy]").forEach(button => {
    button.addEventListener("click", () => {
      const id = button.dataset.addCandy;
      pushSnapshot();
      inventory[id] += 1;
      history.push(`Recebeu 1 doce ${getCandy(id).name.toLowerCase()}.`);
      addCandyDialog.close();
      invalidateSolution();
      refresh();
      showToast(`Adicionado: 1 doce ${getCandy(id).name.toLowerCase()}.`);
    });
  });
}

function renderHistory() {
  if (!history.length) {
    historyList.innerHTML = `<li class="history-empty">Nenhuma troca realizada.</li>`;
    return;
  }

  historyList.innerHTML = history.map(item => `<li>${item}</li>`).join("");
}

function updateButtons() {
  recipesList.querySelectorAll("[data-recipe]").forEach(button => {
    const recipe = RECIPES.find(item => item.id === button.dataset.recipe);
    button.disabled = !canApply(inventory, recipe.input);
  });

  const from = customFrom.value;
  const to = customTo.value;
  customTradeBtn.disabled = from === to || inventory[from] < 3;
  targetTradeBtn.disabled = !hasTarget(inventory);
  undoBtn.disabled = snapshots.length === 0;
}

function updateStatus() {
  const currentScore = calculateScore(inventory);
  const targetScore = calculateScore(TARGET);

  currentScoreEl.textContent = currentScore;
  targetScoreEl.textContent = targetScore;
  totalCandiesEl.textContent = totalCandies(inventory);
  feasibilityBox.className = "feasibility-box";

  if (hasTarget(inventory)) {
    feasibilityBox.classList.add("success");
    feasibilityTitle.textContent = "ARCANA DISPONÍVEL";
    feasibilityText.textContent =
      "Você já possui todos os doces exigidos. O botão RESGATAR está liberado.";
    return;
  }

  if (currentScore < targetScore) {
    feasibilityBox.classList.add("danger");
    feasibilityTitle.textContent = "FALTA VALOR DE TROCA";
    feasibilityText.textContent =
      `O estoque vale ${currentScore} e a Arcana exige ${targetScore}. ` +
      "As receitas atuais não aumentam esse valor.";
    return;
  }

  feasibilityBox.classList.add("warning");
  feasibilityTitle.textContent = "BUSCA RECOMENDADA";
  feasibilityText.textContent =
    "O valor total pode ser suficiente. Use a busca para verificar uma sequência.";
}

function setInventoryManually(id, value) {
  inventory[id] = Math.max(0, Math.floor(value));
  history = [];
  snapshots = [];
  invalidateSolution("Estoque alterado manualmente. Execute uma nova busca.");
  refresh();
}

function pushSnapshot() {
  snapshots.push({
    inventory: { ...inventory },
    history: [...history]
  });
}

function executeTrade(input, output, description) {
  if (!canApply(inventory, input)) {
    showToast("Você não tem os doces necessários.");
    return;
  }

  pushSnapshot();
  inventory = applyDelta(inventory, input, output);
  history.push(description);
  invalidateSolution();
  refresh();
}

function claimTarget() {
  if (!hasTarget(inventory)) {
    showToast("Ainda faltam doces para a Arcana.");
    return;
  }

  pushSnapshot();
  inventory = applyDelta(inventory, TARGET, {});
  history.push("Arcana resgatada: Compass of the Rising Gale.");
  invalidateSolution("Arcana resgatada. Você pode desfazer para continuar simulando.");
  refresh();
  showToast("Arcana resgatada na simulação!");
}

function undo() {
  const previous = snapshots.pop();

  if (!previous) {
    return;
  }

  inventory = previous.inventory;
  history = previous.history;
  invalidateSolution("Última ação desfeita. Execute uma nova busca.");
  refresh();
}

function resetAll() {
  inventory = { ...INITIAL };
  history = [];
  snapshots = [];
  invalidateSolution("Simulação reiniciada. Clique em “Procurar melhor sequência”.");
  refresh();
  showToast("Simulação reiniciada.");
}

function clearHistory() {
  history = [];
  snapshots = [];
  renderHistory();
  updateButtons();
  showToast("Histórico limpo.");
}

function saveState() {
  localStorage.setItem("candyworks-simulator-v2", JSON.stringify({
    inventory,
    history
  }));
  showToast("Estado salvo neste navegador.");
}

function loadState() {
  const saved =
    localStorage.getItem("candyworks-simulator-v2") ||
    localStorage.getItem("candyworks-simulator");

  if (!saved) {
    showToast("Nenhum estado salvo.");
    return;
  }

  try {
    const parsed = JSON.parse(saved);
    inventory = sanitizeState(parsed.inventory);
    history = Array.isArray(parsed.history) ? parsed.history : [];
    snapshots = [];
    invalidateSolution("Estado carregado. Execute uma nova busca.");
    refresh();
    showToast("Estado carregado.");
  } catch {
    showToast("O estado salvo está inválido.");
  }
}

function sanitizeState(state) {
  return Object.fromEntries(
    CANDIES.map(candy => [
      candy.id,
      Math.max(0, Math.floor(Number(state?.[candy.id]) || 0))
    ])
  );
}

function invalidateSolution(
  message = "Estoque alterado. Execute uma nova busca."
) {
  solutionContent.innerHTML = `<p class="muted">${message}</p>`;
}

function showToast(message) {
  toast.textContent = message;
  toast.classList.add("show");
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => toast.classList.remove("show"), 2300);
}

function stateToArray(state) {
  return CANDIES.map(candy => state[candy.id] || 0);
}

function keyOf(values) {
  return values.join(",");
}

function buildSearchActions() {
  const actions = RECIPES.map(recipe => ({
    label: `${recipe.name}: ${humanSide(recipe.input)} → ${humanSide(recipe.output)}`,
    input: stateToArray(recipe.input),
    output: stateToArray(recipe.output)
  }));

  for (const from of CANDIES) {
    for (const to of CANDIES) {
      if (from.id === to.id) continue;

      const input = Object.fromEntries(
        CANDIES.map(candy => [candy.id, candy.id === from.id ? 3 : 0])
      );
      const output = Object.fromEntries(
        CANDIES.map(candy => [candy.id, candy.id === to.id ? 1 : 0])
      );

      actions.push({
        label: `Personalizada: 3 ${from.name.toLowerCase()} → 1 ${to.name.toLowerCase()}`,
        input: stateToArray(input),
        output: stateToArray(output)
      });
    }
  }

  return actions;
}

function canApplyArray(state, input) {
  return state.every((value, index) => value >= input[index]);
}

function applyArray(state, input, output) {
  return state.map(
    (value, index) => value - input[index] + output[index]
  );
}

function meetsTargetArray(state) {
  const target = stateToArray(TARGET);
  return state.every((value, index) => value >= target[index]);
}

function reconstructSolution(goalKey, parents) {
  const steps = [];
  let currentKey = goalKey;

  while (parents.has(currentKey)) {
    const entry = parents.get(currentKey);
    steps.push(entry.action);
    currentKey = entry.previous;
  }

  return steps.reverse();
}

async function searchSolution() {
  const start = stateToArray(inventory);
  const startScore = calculateScore(inventory);
  const targetScore = calculateScore(TARGET);

  solutionContent.innerHTML = `
    <div class="solution-summary">
      <span class="solution-badge warning">ANALISANDO</span>
      <p>Explorando combinações possíveis...</p>
    </div>
  `;

  await new Promise(resolve => setTimeout(resolve, 30));

  if (meetsTargetArray(start)) {
    renderSolution([]);
    return;
  }

  if (startScore < targetScore) {
    renderImpossible(
      `Prova por valor: você tem ${startScore} pontos e precisa de ${targetScore}. ` +
      "Como nenhuma regra configurada aumenta esse valor, não existe sequência possível."
    );
    return;
  }

  const actions = buildSearchActions();
  const queue = [start];
  let cursor = 0;
  const visited = new Set([keyOf(start)]);
  const parents = new Map();
  const MAX_STATES = 250000;

  while (cursor < queue.length && visited.size <= MAX_STATES) {
    const current = queue[cursor++];

    for (const action of actions) {
      if (!canApplyArray(current, action.input)) continue;

      const next = applyArray(current, action.input, action.output);
      const nextKey = keyOf(next);

      if (visited.has(nextKey)) continue;

      visited.add(nextKey);
      parents.set(nextKey, {
        previous: keyOf(current),
        action: action.label
      });

      if (meetsTargetArray(next)) {
        const steps = reconstructSolution(nextKey, parents);
        renderSolution(steps, visited.size);
        return;
      }

      queue.push(next);
    }

    if (cursor % 4000 === 0) {
      solutionContent.innerHTML = `
        <div class="solution-summary">
          <span class="solution-badge warning">ANALISANDO</span>
          <p>${visited.size.toLocaleString("pt-BR")} estados verificados...</p>
        </div>
      `;
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }

  if (visited.size > MAX_STATES) {
    solutionContent.innerHTML = `
      <div class="solution-summary">
        <span class="solution-badge warning">LIMITE ATINGIDO</span>
        <p>
          Foram analisados ${MAX_STATES.toLocaleString("pt-BR")} estados sem solução.
          A busca foi interrompida para evitar travar o navegador.
        </p>
      </div>
    `;
    return;
  }

  renderImpossible(
    `Foram analisados ${visited.size.toLocaleString("pt-BR")} estados possíveis e nenhuma sequência atingiu a meta.`
  );
}

function renderSolution(steps, checkedStates = 1) {
  if (!steps.length) {
    solutionContent.innerHTML = `
      <div class="solution-summary">
        <span class="solution-badge success">META PRONTA</span>
        <p>Você já possui todos os doces necessários para resgatar a Arcana.</p>
      </div>
    `;
    return;
  }

  solutionContent.innerHTML = `
    <div class="solution-summary">
      <span class="solution-badge success">SOLUÇÃO ENCONTRADA</span>
      <p>
        ${steps.length} troca${steps.length === 1 ? "" : "s"}.
        ${checkedStates.toLocaleString("pt-BR")} estados analisados.
      </p>
      <ol class="solution-steps">
        ${steps.map(step => `<li>${step}</li>`).join("")}
      </ol>
    </div>
  `;
}

function renderImpossible(message) {
  solutionContent.innerHTML = `
    <div class="solution-summary">
      <span class="solution-badge danger">SEM SOLUÇÃO</span>
      <p>${message}</p>
    </div>
  `;
}

customFrom.addEventListener("change", updateButtons);
customTo.addEventListener("change", updateButtons);

customTradeBtn.addEventListener("click", () => {
  const from = customFrom.value;
  const to = customTo.value;

  if (from === to) {
    showToast("Escolha doces diferentes.");
    return;
  }

  executeTrade(
    { [from]: 3 },
    { [to]: 1 },
    `Personalizada: 3 ${getCandy(from).name.toLowerCase()} → 1 ${getCandy(to).name.toLowerCase()}`
  );
});

targetTradeBtn.addEventListener("click", claimTarget);
document.querySelector("#undoBtn").addEventListener("click", undo);
document.querySelector("#resetBtn").addEventListener("click", resetAll);
document.querySelector("#clearHistoryBtn").addEventListener("click", clearHistory);
document.querySelector("#saveBtn").addEventListener("click", saveState);
document.querySelector("#loadBtn").addEventListener("click", loadState);
document.querySelector("#searchBtn").addEventListener("click", searchSolution);
document.querySelector("#addOneBtn").addEventListener("click", () => addCandyDialog.showModal());

function refresh() {
  renderInventory();
  renderHistory();
  updateStatus();
  updateButtons();
}

renderTarget();
renderCustomOptions();
renderAddCandyOptions();
renderRecipes();
refresh();
