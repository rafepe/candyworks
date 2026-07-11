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
const undoBtn = document.querySelector("#undoBtn");
const currentScoreEl = document.querySelector("#currentScore");
const targetScoreEl = document.querySelector("#targetScore");
const feasibilityBox = document.querySelector("#feasibilityBox");
const feasibilityTitle = document.querySelector("#feasibilityTitle");
const feasibilityText = document.querySelector("#feasibilityText");
const addCandyDialog = document.querySelector("#addCandyDialog");
const addCandyOptions = document.querySelector("#addCandyOptions");
const toast = document.querySelector("#toast");

function candyIcon(id) {
  return `<span class="candy-icon candy-${id}" aria-hidden="true"></span>`;
}

function getCandy(id) {
  return CANDIES.find(candy => candy.id === id);
}

function calculateScore(state) {
  return CANDIES.reduce((sum, candy) => sum + state[candy.id] * candy.weight, 0);
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

function formatSide(side) {
  return Object.entries(side)
    .filter(([, amount]) => amount > 0)
    .map(([id, amount]) => {
      const candy = getCandy(id);
      return `
        <span class="recipe-candy" title="${amount} ${candy.name}">
          ${candyIcon(id)}
          <span>${amount}</span>
        </span>
      `;
    })
    .join("");
}

function humanSide(side) {
  return Object.entries(side)
    .filter(([, amount]) => amount > 0)
    .map(([id, amount]) => `${amount} ${getCandy(id).name.toLowerCase()}`)
    .join(" + ");
}

function renderInventory() {
  inventoryGrid.innerHTML = CANDIES.map(candy => `
    <article class="candy-card">
      <div class="candy-card-top">
        ${candyIcon(candy.id)}
        <div>
          <strong>${candy.name}</strong>
          <label>Peso ${candy.weight}</label>
        </div>
      </div>

      <div class="counter">
        <button type="button" data-change="-1" data-candy="${candy.id}" aria-label="Diminuir ${candy.name}">−</button>
        <input
          type="number"
          min="0"
          step="1"
          value="${inventory[candy.id]}"
          data-input-candy="${candy.id}"
          aria-label="Quantidade de ${candy.name}"
        >
        <button type="button" data-change="1" data-candy="${candy.id}" aria-label="Aumentar ${candy.name}">+</button>
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

  inventoryGrid.querySelectorAll("[data-input-candy]").forEach(input => {
    input.addEventListener("change", () => {
      const id = input.dataset.inputCandy;
      const value = Math.max(0, Math.floor(Number(input.value) || 0));
      setInventoryManually(id, value);
    });
  });
}

function renderTarget() {
  targetGrid.innerHTML = CANDIES.map(candy => `
    <article class="target-candy">
      ${candyIcon(candy.id)}
      <span>${candy.name}</span>
      <span class="target-count">${TARGET[candy.id]}</span>
    </article>
  `).join("");
}

function renderRecipes() {
  recipesList.innerHTML = RECIPES.map(recipe => `
    <article class="recipe-card">
      <h3>${recipe.name}</h3>

      <div class="recipe-flow">
        <div class="recipe-side">${formatSide(recipe.input)}</div>
        <span class="recipe-arrow">→</span>
        <div class="recipe-side">${formatSide(recipe.output)}</div>
      </div>

      <button class="btn btn-secondary" data-recipe="${recipe.id}">
        Aplicar troca
      </button>
    </article>
  `).join("");

  recipesList.querySelectorAll("[data-recipe]").forEach(button => {
    button.addEventListener("click", () => {
      const recipe = RECIPES.find(item => item.id === button.dataset.recipe);
      executeTrade(recipe.input, recipe.output, `${recipe.name}: ${humanSide(recipe.input)} → ${humanSide(recipe.output)}`);
    });
  });

  updateRecipeButtons();
}

function renderCustomOptions() {
  const options = CANDIES.map(candy => `<option value="${candy.id}">${candy.name}</option>`).join("");
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
      snapshots.push({
        inventory: { ...inventory },
        history: [...history]
      });
      inventory[id] += 1;
      history.push(`Cenário: +1 doce ${getCandy(id).name.toLowerCase()}.`);
      addCandyDialog.close();
      refresh();
      showToast(`Adicionado 1 doce ${getCandy(id).name.toLowerCase()}.`);
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

function updateRecipeButtons() {
  recipesList.querySelectorAll("[data-recipe]").forEach(button => {
    const recipe = RECIPES.find(item => item.id === button.dataset.recipe);
    button.disabled = !canApply(inventory, recipe.input);
  });

  const from = customFrom.value;
  const to = customTo.value;
  customTradeBtn.disabled = from === to || inventory[from] < 3;
}

function updateScoresAndFeasibility() {
  const currentScore = calculateScore(inventory);
  const targetScore = calculateScore(TARGET);
  currentScoreEl.textContent = currentScore;
  targetScoreEl.textContent = targetScore;

  feasibilityBox.className = "feasibility-box";

  if (hasTarget(inventory)) {
    feasibilityBox.classList.add("success");
    feasibilityTitle.textContent = "Meta alcançada";
    feasibilityText.textContent = "Seu estoque já contém todos os doces necessários para pegar a Arcana.";
    return;
  }

  if (currentScore < targetScore) {
    feasibilityBox.classList.add("danger");
    feasibilityTitle.textContent = "Impossível com este estoque";
    feasibilityText.textContent =
      `Seu estoque vale ${currentScore} pontos e a Arcana exige ${targetScore}. ` +
      "Nenhuma receita configurada aumenta esse valor, então infinitas trocas não resolvem.";
    return;
  }

  feasibilityBox.classList.add("warning");
  feasibilityTitle.textContent = "Pode ser possível";
  feasibilityText.textContent =
    "O valor total não impede a solução. Use a busca automática para verificar as combinações.";
}

function setInventoryManually(id, value) {
  inventory[id] = Math.max(0, Math.floor(value));
  history = [];
  snapshots = [];
  solutionContent.innerHTML = `<p class="muted">Estoque alterado. Execute uma nova busca.</p>`;
  refresh();
}

function executeTrade(input, output, description) {
  if (!canApply(inventory, input)) {
    showToast("Você não tem os doces necessários.");
    return;
  }

  snapshots.push({
    inventory: { ...inventory },
    history: [...history]
  });

  inventory = applyDelta(inventory, input, output);
  history.push(description);
  solutionContent.innerHTML = `<p class="muted">Estoque alterado. Execute uma nova busca.</p>`;
  refresh();
}

function undo() {
  const previous = snapshots.pop();

  if (!previous) {
    return;
  }

  inventory = previous.inventory;
  history = previous.history;
  solutionContent.innerHTML = `<p class="muted">Troca desfeita. Execute uma nova busca.</p>`;
  refresh();
}

function resetAll() {
  inventory = { ...INITIAL };
  history = [];
  snapshots = [];
  solutionContent.innerHTML = `<p class="muted">Clique em “Procurar sequência automaticamente”.</p>`;
  refresh();
  showToast("Simulador reiniciado.");
}

function clearHistory() {
  history = [];
  snapshots = [];
  renderHistory();
  undoBtn.disabled = true;
  showToast("Histórico limpo.");
}

function saveState() {
  localStorage.setItem("candyworks-simulator", JSON.stringify({
    inventory,
    history
  }));
  showToast("Estado salvo neste navegador.");
}

function loadState() {
  const saved = localStorage.getItem("candyworks-simulator");

  if (!saved) {
    showToast("Nenhum estado salvo.");
    return;
  }

  try {
    const parsed = JSON.parse(saved);
    inventory = sanitizeState(parsed.inventory);
    history = Array.isArray(parsed.history) ? parsed.history : [];
    snapshots = [];
    solutionContent.innerHTML = `<p class="muted">Estado carregado. Execute uma nova busca.</p>`;
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

function showToast(message) {
  toast.textContent = message;
  toast.classList.add("show");
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => toast.classList.remove("show"), 2200);
}

function stateToArray(state) {
  return CANDIES.map(candy => state[candy.id]);
}

function arrayToState(values) {
  return Object.fromEntries(CANDIES.map((candy, index) => [candy.id, values[index]]));
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

      const input = Object.fromEntries(CANDIES.map(candy => [candy.id, candy.id === from.id ? 3 : 0]));
      const output = Object.fromEntries(CANDIES.map(candy => [candy.id, candy.id === to.id ? 1 : 0]));

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
  return state.map((value, index) => value - input[index] + output[index]);
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
      <span class="solution-badge warning">Analisando</span>
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
      "Como nenhuma regra aumenta esse valor, não existe sequência possível."
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
          <span class="solution-badge warning">Analisando</span>
          <p>${visited.size.toLocaleString("pt-BR")} estados verificados...</p>
        </div>
      `;
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }

  if (visited.size > MAX_STATES) {
    solutionContent.innerHTML = `
      <div class="solution-summary">
        <span class="solution-badge warning">Limite atingido</span>
        <p>
          Foram analisados ${MAX_STATES.toLocaleString("pt-BR")} estados sem encontrar uma solução.
          O navegador interrompeu a busca para evitar travamento.
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
        <span class="solution-badge success">Meta pronta</span>
        <p>Você já possui todos os doces necessários.</p>
      </div>
    `;
    return;
  }

  solutionContent.innerHTML = `
    <div class="solution-summary">
      <span class="solution-badge success">Solução encontrada</span>
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
      <span class="solution-badge danger">Sem solução</span>
      <p>${message}</p>
    </div>
  `;
}

function refresh() {
  renderInventory();
  renderHistory();
  updateScoresAndFeasibility();
  updateRecipeButtons();
  undoBtn.disabled = snapshots.length === 0;
}

customFrom.addEventListener("change", updateRecipeButtons);
customTo.addEventListener("change", updateRecipeButtons);

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

document.querySelector("#undoBtn").addEventListener("click", undo);
document.querySelector("#resetBtn").addEventListener("click", resetAll);
document.querySelector("#clearHistoryBtn").addEventListener("click", clearHistory);
document.querySelector("#saveBtn").addEventListener("click", saveState);
document.querySelector("#loadBtn").addEventListener("click", loadState);
document.querySelector("#searchBtn").addEventListener("click", searchSolution);
document.querySelector("#addOneBtn").addEventListener("click", () => addCandyDialog.showModal());

renderTarget();
renderCustomOptions();
renderAddCandyOptions();
renderRecipes();
refresh();
