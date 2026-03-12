# ⚙️ RPA Framework — Automatisation Windows & Desktop

RPA Framework (rpaframework) est la collection de bibliothèques open-source
de Robocorp, conçue pour Robot Framework et Python. Elle fournit des librairies
prêtes à l'emploi pour automatiser des tâches Windows : contrôle d'applications,
interactions clavier/souris, gestion de fichiers, capture d'écran, OCR.

Ce projet explore les capacités de `rpaframework-windows` pour l'automatisation
d'applications desktop et la mise en place d'une pipeline CI/CD via GitHub Actions.

---

## Bibliothèques utilisées

```
  rpaframework
  │
  ├── RPA.Windows          ← contrôle d'applications Windows natives
  ├── RPA.Desktop          ← interactions clavier, souris, screenshot
  ├── RPA.FileSystem       ← lecture/écriture/manipulation de fichiers
  ├── RPA.PDF              ← extraction et génération de PDFs
  └── RPA.Robocorp.WorkItems  ← gestion des données d'entrée/sortie
```

---

## Exemple : automatisation d'un formulaire Windows

```robotframework
*** Settings ***
Library    RPA.Windows
Library    RPA.Desktop
Library    RPA.FileSystem

Suite Setup     Open Target Application
Suite Teardown  Close Target Application

*** Variables ***
${APP}    notepad.exe
${OUTPUT_DIR}    ${CURDIR}/results

*** Test Cases ***

Saisir et sauvegarder un document
    [Documentation]    Automatise la saisie de texte dans Notepad et la sauvegarde
    # Trouver la zone de texte et y écrire
    ${editor}=    Get Element    name:Text Editor
    Type Text     ${editor}    Rapport de test automatisé\nDate: ${TODAY}

    # Sauvegarder via raccourci clavier
    Send Keys    ctrl+s
    Wait For Element    name:Save As    timeout=5

    # Remplir le nom de fichier dans la boîte de dialogue
    ${filename_field}=    Get Element    name:File name
    Clear Value     ${filename_field}
    Type Text       ${filename_field}    rapport_test_${TODAY}.txt
    Click Element   name:Save

    # Vérifier que le fichier existe
    File Should Exist    ${OUTPUT_DIR}/rapport_test_${TODAY}.txt

Capturer l état de l interface
    [Documentation]    Screenshot automatique pour documentation et débogage
    ${screenshot}=    Take Screenshot    name=etat_interface
    Log    Screenshot sauvegardé : ${screenshot}    level=INFO

Remplir un formulaire avec données externes
    [Documentation]    Lit les données de test depuis un CSV et remplit le formulaire
    ${test_data}=    Read CSV    test_data/formulaire_cas.csv
    FOR    ${row}    IN    @{test_data}
        Open Application    ${APP}
        Fill Form Field     name:field_nom       ${row}[nom]
        Fill Form Field     name:field_email     ${row}[email]
        Select Dropdown     name:combo_role      ${row}[role]
        Click Element       name:btn_valider
        Verify Confirmation    ${row}[message_attendu]
        Close Application
    END

*** Keywords ***
Open Target Application
    Windows Run    ${APP}
    Wait For Element    name:Text Editor    timeout=10

Close Target Application
    Close Window    name:Notepad

Fill Form Field
    [Arguments]    ${locator}    ${value}
    ${element}=    Get Element    ${locator}
    Clear Value    ${element}
    Type Text      ${element}    ${value}

Verify Confirmation
    [Arguments]    ${expected_message}
    ${actual}=    Get Value    name:label_confirmation
    Should Be Equal    ${actual}    ${expected_message}
```

---

## Pipeline CI/CD GitHub Actions

```yaml
# .github/workflows/robot-tests.yml
name: Robot Framework Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  robot-tests:
    runs-on: windows-latest   # Requis pour WinAppDriver / RPA.Windows

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install robotframework
          pip install rpaframework
          pip install rpaframework-windows

      - name: Run Robot Framework tests
        run: |
          robot --outputdir results --loglevel DEBUG tests/

      - name: Publish Test Report
        uses: actions/upload-artifact@v4
        if: always()   # Publier même si les tests échouent
        with:
          name: robot-test-results
          path: results/
          # → log.html, report.html, output.xml disponibles dans les artifacts
```

---

## Rapport de test Robot Framework

Après chaque exécution, Robot Framework génère automatiquement :

```
  results/
  ├── report.html    ← résumé visuel (PASS/FAIL, durée, statistiques)
  ├── log.html       ← détail complet de chaque étape + screenshots
  └── output.xml     ← données brutes pour intégration CI/CD (Allure, etc.)
```

---

## Ce que j'ai appris

`RPA.Windows` utilise le même moteur UIAutomation que WinAppDriver, mais avec
une API plus haut niveau et des mots-clés plus expressifs. Le compromis :
moins de contrôle sur les locators bas niveau, mais des tests plus lisibles
et plus rapides à écrire.

Pour des applications .NET bien construites (avec AutomationId sur les contrôles),
`RPA.Windows` est souvent suffisant. Pour des applications plus complexes ou avec
des contrôles tiers mal accessibles, WinAppDriver + DesktopLibrary donne plus
de flexibilité. Les deux approches se complètent.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
