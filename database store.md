import os
import sys
import logging
import argparse
import pandas as pd
from sqlalchemy import create_engine, text
from sklearn.ensemble import IsolationForest


def setup_logging(verbose: bool):
    logging.basicConfig(
        level=logging.DEBUG if verbose else logging.INFO,
        format="%(asctime)s %(levelname)s %(message)s"
    )


def get_engine():
    dsn = os.getenv("PG_DSN", "postgresql://postgres:swosti@localhost:5432/wargame")
    logging.info(f"Connecting to database with DSN: {dsn}")
    return create_engine(dsn, pool_pre_ping=True, future=True)


def fetch_data(engine, table: str, time_col: str | None, hours: int | None):
    query = "SELECT id, device_id, temperature, humidity"
    if time_col:
        query += f", {time_col}"
    query += f" FROM {table}"
    if time_col and hours and hours > 0:
        query += f" WHERE {time_col} >= NOW() - INTERVAL '{hours} hour'"
    logging.debug(f"Executing SQL: {query}")
    return pd.read_sql(text(query), engine)


def detect_anomalies(df: pd.DataFrame, contamination: float, seed: int, train_on_normals: bool):
    if df.empty:
        logging.warning("No data returned from source table.")
        return df, pd.DataFrame()

    features = df[["temperature", "humidity"]].astype(float).dropna()
    df_clean = df.loc[features.index].copy()

    if features.empty:
        logging.warning("No valid feature rows after dropping NA.")
        return df, pd.DataFrame()

    model = IsolationForest(contamination=contamination, random_state=seed)

    if train_on_normals:
        q_low = features.quantile(0.05)
        q_high = features.quantile(0.95)
        mask = ((features >= q_low) & (features <= q_high)).all(axis=1)
        train_X = features[mask]
        if len(train_X) < 10:
            logging.warning("Insufficient presumed-normal samples; training on all data.")
            train_X = features
        model.fit(train_X)
        preds = model.predict(features)
    else:
        preds = model.fit_predict(features)

    df_clean["anomaly"] = preds
    df_clean["anomaly_label"] = df_clean["anomaly"].map({1: "normal", -1: "anomaly"})

    anomalies = df_clean[df_clean["anomaly_label"] == "anomaly"].copy()
    return df_clean, anomalies


def insert_anomalies(engine, anomalies: pd.DataFrame, target_table: str = "anomalies"):
    if anomalies.empty:
        logging.info("No anomalies to insert.")
        return 0

    rows = [
        {
            "device_id": r["device_id"],
            "temperature": float(r["temperature"]),
            "humidity": float(r["humidity"]),
        }
        for _, r in anomalies.iterrows()
    ]

    sql = text(
        """
        INSERT INTO anomalies (device_id, temperature, humidity)
        VALUES (:device_id, :temperature, :humidity)
        """
    )

    with engine.begin() as conn:
        conn.execute(sql, rows)

    return len(rows)


def parse_args():
    parser = argparse.ArgumentParser(description="Detect anomalies using IsolationForest.")
    parser.add_argument("--source-table", default="sensor_data", help="Source table name")
    parser.add_argument("--time-col", default=None, help="Timestamp column in source table")
    parser.add_argument("--hours", type=int, default=None, help="Limit to last N hours")
    parser.add_argument("--contamination", type=float, default=0.05, help="IsolationForest contamination")
    parser.add_argument("--seed", type=int, default=42, help="Random seed for reproducibility")
    parser.add_argument("--train-on-normals", action="store_true", help="Train on middle quantiles as normal")
    parser.add_argument("--verbose", action="store_true", help="Enable verbose logging")
    return parser.parse_args()


def main():
    args = parse_args()
    setup_logging(args.verbose)
    logging.debug(f"Using Python executable: {sys.executable}")

    try:
        engine = get_engine()
        df = fetch_data(engine, args.source_table, args.time_col, args.hours)
        df_labelled, anomalies = detect_anomalies(
            df, args.contamination, args.seed, args.train_on_normals
        )
        if not df_labelled.empty:
            logging.info("Sample labeled rows:\n%s", df_labelled.tail(5).to_string(index=False))
        inserted_rows = insert_anomalies(engine, anomalies)
        if inserted_rows:
            logging.info(f"Inserted {inserted_rows} anomalies into the database.")
        else:
            logging.info("No anomalies found to insert.")
    except Exception as exc:
        logging.exception(f"Job failed due to: {exc}")
        sys.exit(1)


if __name__ == "__main__":
    main()
