## Fraudfinder : une série d'ateliers complets sur la façon de créer un système de détection des fraudes en temps réel sur Google Cloud.

[Fraudfinder](https://github.com/googlecloudplatform/fraudfinder) FraudFinder est un atelier Gold Data to AI pour montrer une architecture de bout en bout, des données brutes au MLOps, en passant par le cas d'utilisation de la détection de fraude en temps réel. Fraudfinder est une série d'ateliers visant à présenter le parcours complet des données vers l'IA sur Google Cloud, à travers le cas d'utilisation de la détection des fraudes en temps réel. Tout au long des ateliers Fraudfinder, vous apprendrez à lire les données historiques des transactions de paiement stockées dans un entrepôt de données, à lire un flux en direct de nouvelles transactions, à effectuer une analyse exploratoire des données (EDA), à effectuer l'ingénierie des fonctionnalités, à ingérer des fonctionnalités dans un magasin de fonctionnalités, à former un modèle à l'aide du magasin de fonctionnalités, enregistrez votre modèle dans un registre de modèles, évaluez votre modèle, déployez votre modèle sur un point de terminaison, effectuez une inférence en temps réel sur votre modèle avec le magasin de fonctionnalités et surveillez votre modèle. Ci-dessous vous trouverez une image de l'architecture globale :

![image](./misc/images/fraudfinder-architecture.png)

## Comment utiliser ce dépôt

Ce référentiel est organisé en différents notebooks comme :

* [00_environment_setup.ipynb](00_environment_setup.ipynb)
* [01_exploratory_data_analysis.ipynb](01_exploratory_data_analysis.ipynb)
* [02_feature_engineering_batch.ipynb](02_feature_engineering_batch.ipynb)
* [03_feature_engineering_streaming.ipynb](03_feature_engineering_streaming.ipynb)
* [bqml/](bqml/)
  * [04_model_training_and_prediction.ipynb](bqml/04_model_training_and_prediction.ipynb)
  * [05_model_training_pipeline_formalization.ipynb](bqml/05_model_training_pipeline_formalization.ipynb)
  * [06_model_monitoring.ipynb](bqml/06_model_monitoring.ipynb)

## Créer un projet Google Cloud
Avant de commencer, il est recommandé de créer un nouveau projet Google Cloud afin que les activités de cet atelier n'interfèrent pas avec d'autres projets existants.

Si vous utilisez un compte temporaire fourni, veuillez simplement sélectionner un projet existant pré-créé avant l'événement, comme indiqué dans l'image ci-dessous.

Il n'est pas rare que le projet pré-créé dans le compte temporaire fourni porte un nom différent. Veuillez vérifier auprès du fournisseur de compte si vous avez besoin de plus de précisions sur le projet à choisir.

Si vous n'utilisez PAS de compte temporaire, veuillez créer un nouveau projet Google Cloud et sélectionner ce projet. Vous pouvez vous référer à la documentation officielle  ([Creating and Managing Projects](https://cloud.google.com/resource-manager/docs/creating-managing-projects)) pour des instructions détaillées.

## Exécuter les cahiers
Pour exécuter les notebooks avec succès, suivez les étapes ci-dessous.

### Étape 0 : Sélectionnez votre projet Google Cloud
Veuillez vous assurer que vous avez sélectionné un projet Google Cloud comme indiqué ci-dessous :
  ![image](./misc/images/select-project-dasher.png)

### Étape 1 : Configuration initiale à l'aide de Cloud Shell
- Activez Cloud Shell dans votre projet en cliquant sur le bouton`Activate Cloud Shell` comme indiqué dans l'image ci-dessous.
  ![image](./misc/images/activate-cloud-shell.png)

- Une fois Cloud Shell activé, copiez les codes suivants et exécutez-les dans Cloud Shell pour activer les API nécessaires, puis créez des abonnements Pub/Sub pour lire les transactions en streaming à partir de sujets Pub/Sub publics.

- Autorisez Cloud Shell s'il vous y invite. Veuillez noter que cette étape peut prendre quelques minutes. Vous pouvez accéder à la [Pub/Sub console](https://console.cloud.google.com/cloudpubsub/subscription/) pour vérifier les abonnements.

 

  ```shell
  gcloud services enable notebooks.googleapis.com
  gcloud services enable cloudresourcemanager.googleapis.com
  gcloud services enable aiplatform.googleapis.com
  gcloud services enable pubsub.googleapis.com
  gcloud services enable run.googleapis.com
  gcloud services enable cloudbuild.googleapis.com
  gcloud services enable dataflow.googleapis.com
  gcloud services enable bigquery.googleapis.com
  gcloud services enable artifactregistry.googleapis.com
  gcloud services enable iam.googleapis.com
  
  gcloud pubsub subscriptions create "ff-tx-sub" --topic="ff-tx" --topic-project="cymbal-fraudfinder"
  gcloud pubsub subscriptions create "ff-tx-for-feat-eng-sub" --topic="ff-tx" --topic-project="cymbal-fraudfinder"
  gcloud pubsub subscriptions create "ff-txlabels-sub" --topic="ff-txlabels" --topic-project="cymbal-fraudfinder"
  
  # Run the following command to grant the Compute Engine default service account access to read and write pipeline artifacts in Google Cloud Storage.
  PROJECT_ID=$(gcloud config get-value project)
  PROJECT_NUM=$(gcloud projects list --filter="$PROJECT_ID" --format="value(PROJECT_NUMBER)")
  gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:${PROJECT_NUM}-compute@developer.gserviceaccount.com"\
        --role='roles/storage.admin'
  gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:${PROJECT_NUM}@cloudbuild.gserviceaccount.com"\
        --role='roles/aiplatform.admin'
  gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:$PROJECT_NUM-compute@developer.gserviceaccount.com"\
        --role='roles/run.admin'
  gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:$PROJECT_NUM-compute@developer.gserviceaccount.com"\
        --role='roles/resourcemanager.projectIamAdmin'
  gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:service-${PROJECT_NUM}@gcp-sa-aiplatform.iam.gserviceaccount.com"\
        --role='roles/artifactregistry.writer'
  gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:service-${PROJECT_NUM}@gcp-sa-aiplatform.iam.gserviceaccount.com"\
        --role='roles/storage.objectAdmin'   
  ```

#### Étape 2 : Créer une instance de notebook géré par l'utilisateur sur Vertex AI Workbench
- Accédez à la page [Vertex AI Workbench](https://console.cloud.google.com/vertex-ai/workbench/list/instances), cliquez sur"**USER-MANAGED NOTEBOOKS**" et cliquez sur "**+ NEW NOTEBOOK**" comme indiqué dans l'image ci-dessous.
  ![image](./misc/images/click-new-notebook.png)
  
- Veuillez vous assurer d'avoir sélectionné le bon projet lors de la création d'un nouveau bloc-notes. En cliquant sur "**+ NEW NOTEBOOK**", une liste d'options d'instance de bloc-notes vous sera présentée. SélectionnerPython 3
  ![image](./misc/images/select-notebook-instance.png)

- Choisissez un nom (ou laissez-le par défaut), sélectionnez un emplacement, puis cliquez sur  "**CREATE**" pour créer l'instance de notebook.
  ![image](./misc/images/create-notebook-instance.png)

- L'instance sera prête lorsque vous verrez une coche verte et pourrez cliquer sur "**OPEN JUPYTERLAB**" sur la [User-Managed Notebooks page](https://console.cloud.google.com/vertex-ai/workbench/list/instances). Cela peut prendre quelques minutes pour que l'instance soit prête.
  ![image](./misc/images/notebook-instance-ready.png)

#### Step 3: Open JupyterLab
- Cliquez sur "**OPEN JUPYTERLAB**", ce qui devrait lancer votre notebook géré dans un nouvel onglet.

#### Étape 4 : Ouverture d'un terminal
- Ouvrez un terminal via le menu fichier : **File > New > Terminal**.
  ![image](./misc/images/file-new-terminal.png)
  ![image](./misc/images/terminal.png)

#### tape 5 : Cloner ce dépôt
- Exécutez le code suivant pour cloner ce dépôt :
  ```
  git clone https://github.com/GoogleCloudPlatform/fraudfinder.git
  ```

- Vous pouvez également accéder au menu en haut à gauche de l'environnement Jupyter Lab et cliquer sur **Git > Clone a repository**.

- Une fois cloné, vous devriez maintenant voir le dossier **fraudfinder** dans votre répertoire principal.
  ![image](./misc/images/git-clone-on-terminal.png)


#### Étape 6 : Ouvrez le premier bloc-notes(notebook)

- Ouvrez le premier notebook: `00_environment_setup.ipynb`
  ![image](./misc/images/open-notebook-00.png)

- Suivez les instructions du cahier et continuez avec les cahiers (notebooks) restants. 
