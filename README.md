import os
import pandas as pd
import numpy as np
import spacy
import nltk
from typing import Dict, List, Any

import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import RandomForestRegressor
from sklearn.pipeline import Pipeline
from sklearn.metrics import mean_absolute_error, r2_score, mean_squared_error

class OptimizedNewsArticleAnalyzer:
    def __init__(self, dataset_path: str = None):
        """
        Initialize the NLP analysis tool
        - Load NLP resources (language processing tools)
        - Create a sample dataset for analysis
        """
        # Load language processing resources
        self._load_nlp_resources()
        
        # Create a sample dataset to work with
        self.df = self._create_sample_dataset()

    def _load_nlp_resources(self):
        """
        Prepare language processing tools
        - Download necessary linguistic resources
        - Load SpaCy for advanced text processing
        - Set up stop words (common words to ignore)
        """
        try:
            # Download language processing resources
            nltk.download('punkt', quiet=True)
            nltk.download('stopwords', quiet=True)
            
            # Load SpaCy's language model
            self.nlp = spacy.load('en_core_web_sm')
            
            # Create a set of stop words to filter out common, less meaningful words
            self.stop_words = set(nltk.corpus.stopwords.words('english'))
        except Exception as e:
            print(f"NLP Resource Loading Error: {e}")
            self.nlp = None
            self.stop_words = set()

    @staticmethod
    def _create_sample_dataset() -> pd.DataFrame:
        """
        Create a sample dataset for testing and demonstration
        - Provides a variety of text samples
        - Includes different metrics like likes and shares
        
        Returns:
            DataFrame with text, likes, and shares
        """
        return pd.DataFrame({
            'text': [
                'Apple Inc. is a technology company in Cupertino.',
                'Google was founded by Larry Page and Sergey Brin in California.',
                'Microsoft is headquartered in Washington state.',
                'Facebook is a social media platform created by Mark Zuckerberg.',
                'Amazon is a global e-commerce and cloud computing company.',
                'Netflix revolutionized the streaming entertainment industry.',
                'Tesla is leading the electric vehicle revolution.',
                'SpaceX is pushing the boundaries of space exploration.',
                'Uber transformed the transportation industry.',
                'Airbnb changed the way people travel and find accommodations.'
            ],
            'likes': [100, 250, 150, 180, 220, 90, 130, 200, 170, 140],
            'shares': [50, 120, 80, 90, 110, 45, 65, 100, 85, 70]
        })

    def preprocess_text(self, text: str) -> str:
        """
        Clean and prepare text for analysis
        - Convert to lowercase
        - Remove special characters
        - Remove stop words
        - Tokenize the text
        
        Args:
            text: Input text to preprocess
        
        Returns:
            Cleaned and processed text
        """
        # Handle empty or null inputs
        if pd.isna(text):
            return ""
        
        # Convert to string and lowercase
        text = str(text).lower()
        
        # Remove special characters, keep only alphanumeric
        text = ''.join(char for char in text if char.isalnum() or char.isspace())
        
        # Use SpaCy for advanced tokenization and stop word removal
        doc = self.nlp(text)
        tokens = [token.text for token in doc if not token.is_stop]
        
        return ' '.join(tokens)

    def extract_named_entities(self, text: str) -> Dict[str, List[str]]:
        """
        Extract important named entities from text
        - Identify organizations
        - Identify geographical locations
        - Identify person names
        
        Args:
            text: Input text to extract entities from
        
        Returns:
            Dictionary of entity types and their occurrences
        """
        # Process text with SpaCy's named entity recognition
        doc = self.nlp(str(text))
        
        # Initialize entity categories
        entities = {
            'ORG': [],      # Organizations
            'GPE': [],      # Geographical locations
            'PERSON': []    # Person names
        }
        
        # Extract and categorize entities
        for ent in doc.ents:
            if ent.label_ in entities:
                entities[ent.label_].append(ent.text)
        
        return entities

    def engineer_features(self) -> pd.DataFrame:
        """
        Create advanced features from raw text data
        - Preprocess text
        - Extract named entities
        - Count different types of entities
        - Calculate article length
        
        Returns:
            DataFrame with engineered features
        """
        # Preprocess text
        self.df['cleaned_text'] = self.df['text'].apply(self.preprocess_text)
        
        # Extract named entities
        self.df['named_entities'] = self.df['text'].apply(self.extract_named_entities)
        
        # Count different types of entities
        self.df['org_count'] = self.df['named_entities'].apply(lambda x: len(x['ORG']))
        self.df['gpe_count'] = self.df['named_entities'].apply(lambda x: len(x['GPE']))
        self.df['person_count'] = self.df['named_entities'].apply(lambda x: len(x['PERSON']))
        
        # Calculate article length
        self.df['article_length'] = self.df['cleaned_text'].str.split().str.len()
        
        return self.df

    def train_predictive_model(self, target_column: str = 'likes') -> Dict[str, Any]:
        """
        Build a machine learning model to predict engagement
        - Prepare features
        - Split data into training and testing sets
        - Scale features
        - Train Random Forest Regressor
        - Evaluate model performance
        
        Args:
            target_column: Column to predict (default is 'likes')
        
        Returns:
            Dictionary of model performance metrics
        """
        # Select features for prediction
        features = ['org_count', 'gpe_count', 'person_count', 'article_length']
        
        # Prepare data
        X = self.df[features]
        y = self.df[target_column]
        
        # Create a machine learning pipeline
        pipeline = Pipeline([
            ('scaler', StandardScaler()),  # Normalize features
            ('regressor', RandomForestRegressor(n_estimators=100, random_state=42))
        ])
        
        # Perform cross-validation to get robust performance estimates
        mae_scores = cross_val_score(pipeline, X, y, cv=5, scoring='neg_mean_absolute_error')
        r2_scores = cross_val_score(pipeline, X, y, cv=5, scoring='r2')
        
        # Split data into training and testing sets
        X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
        
        # Train the final model
        pipeline.fit(X_train, y_train)
        
        # Make predictions and calculate metrics
        y_pred = pipeline.predict(X_test)
        
        return {
            'model': pipeline,
            'mae': mean_absolute_error(y_test, y_pred),  # Mean Absolute Error
            'mse': mean_squared_error(y_test, y_pred),   # Mean Squared Error
            'r2': r2_score(y_test, y_pred),              # R-squared Score
            'cv_mae': -mae_scores.mean(),                # Cross-validated MAE
            'cv_r2': r2_scores.mean()                    # Cross-validated R-squared
        }

    def visualize_results(self):
        """
        Create visualizations to understand the data
        - Entity frequency bar chart
        - Feature correlation heatmap
        - Scatter plot of organizations vs article length
        """
        # Create a figure with three subplots
        fig, axes = plt.subplots(1, 3, figsize=(20, 6))
        
        # Plot 1: Entity Frequency Bar Chart
        entity_counts = self.df[['org_count', 'gpe_count', 'person_count']].sum()
        sns.barplot(x=entity_counts.index, y=entity_counts.values, ax=axes[0])
        axes[0].set_title('Entity Type Frequency')
        axes[0].set_xlabel('Entity Types')
        axes[0].set_ylabel('Total Count')
        
        # Plot 2: Feature Correlation Heatmap
        correlation_columns = ['org_count', 'gpe_count', 'person_count', 'article_length']
        correlation_matrix = self.df[correlation_columns].corr()
        sns.heatmap(correlation_matrix, annot=True, cmap='coolwarm', ax=axes[1])
        axes[1].set_title('Feature Correlation')
        
        # Plot 3: Scatter Plot of Organizations vs Article Length
        sns.scatterplot(data=self.df, x='org_count', y='article_length', 
                        hue='likes', palette='viridis', ax=axes[2])
        axes[2].set_title('Organizations vs Article Length')
        
        # Adjust layout and display plots
        plt.tight_layout()
        plt.show()

def main():
    """
    Main execution function
    - Initialize the analyzer
    - Engineer features
    - Train predictive model
    - Print model performance
    - Visualize results
    """
    # Create an instance of the analyzer
    analyzer = OptimizedNewsArticleAnalyzer()
    
    # Perform feature engineering
    featured_df = analyzer.engineer_features()
    
    # Train the predictive model
    model_results = analyzer.train_predictive_model()
    
    # Print model performance metrics
    for metric, value in model_results.items():
        if metric != 'model':
            print(f"{metric}: {value}")
    
    # Create visualizations
    analyzer.visualize_results()

# Ensure the script only runs when directly executed
if __name__ == "__main__":
    main()
