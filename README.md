from flask import Flask, jsonify
import numpy as np
import pandas as pd
import datetime as dt
from sqlalchemy import create_engine, func, inspect
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session

# FlaskApp
app = Flask(__name__)

# Create Engine
engine = create_engine("sqlite:///C:/Users/TeddySong/Desktop/sqlalchemy-challenge/Resources/hawaii.sqlite")


# Reflect an existing database into a new model
Base = automap_base()
# reflect the tables
Base.prepare(autoload_with=engine)

# Save reference to the table
Measurement = Base.classes.measurement
Station = Base.classes.station

# Create Session
session = Session(engine)

# Flask Routes
@app.route("/")
def welcome():
    return (
        f"Welcome to the Climate API!<br/>"
        f"Available Routes:<br/>"
        f"/api/v1.0/precipitation<br/>"
        f"/api/v1.0/stations<br/>"
        f"/api/v1.0/tobs<br/>"
        f"/api/v1.0/<start><br/>"
        f"/api/v1.0/<start>/<end>"
    )

# /api/v1.0/precipitation routes
@app.route("/api/v1.0/precipitation")
def precipitation():
    # recent dates
    most_recent_date = session.query(func.max(Measurement.date)).scalar()
    most_recent_date = dt.datetime.strptime(most_recent_date, '%Y-%m-%d')

    # Calculate 12 month ago data
    one_year_ago = most_recent_date - dt.timedelta(days=365)

    # Search precip data
    precip_data = session.query(Measurement.date, Measurement.prcp).\
        filter(Measurement.date >= one_year_ago).all()

    # Convert to dict
    precip_dict = {date: prcp for date, prcp in precip_data}

    return jsonify(precip_dict)

# /api/v1.0/stations route
@app.route("/api/v1.0/stations")
def stations():
    # search session
    results = session.query(Station.station).all()

    # convert to list
    stations_list = list(np.ravel(results))

    return jsonify(stations_list)

# /api/v1.0/tobs route
@app.route("/api/v1.0/tobs")
def tobs():
    # most active station
    most_active_station = session.query(Measurement.station, func.count(Measurement.station)).\
        group_by(Measurement.station).\
        order_by(func.count(Measurement.station).desc()).first()[0]

    # most recent data in past 12 months
    most_recent_date = session.query(func.max(Measurement.date)).scalar()
    most_recent_date = dt.datetime.strptime(most_recent_date, '%Y-%m-%d')
    one_year_ago = most_recent_date - dt.timedelta(days=365)

    temperature_data = session.query(Measurement.date, Measurement.tobs).\
        filter(Measurement.station == most_active_station).\
        filter(Measurement.date >= one_year_ago).all()

    # convert to list
    tobs_list = list(np.ravel(temperature_data))

    return jsonify(tobs_list)

# /api/v1.0/<start> route
@app.route("/api/v1.0/<start>")
def start_date(start):
    # converte to data time
    start_date = dt.datetime.strptime(start, '%Y-%m-%d')

    # TMIN、TAVG & TMAX
    results = session.query(func.min(Measurement.tobs),
                            func.avg(Measurement.tobs),
                            func.max(Measurement.tobs)).\
        filter(Measurement.date >= start_date).all()

    # Results
    temp_data = {
        "TMIN": results[0][0],
        "TAVG": results[0][1],
        "TMAX": results[0][2]
    }

    return jsonify(temp_data)

# /api/v1.0/<start>/<end> Route
@app.route("/api/v1.0/<start>/<end>")
def start_end_date(start, end):
    # Convert to datatime
    start_date = dt.datetime.strptime(start, '%Y-%m-%d')
    end_date = dt.datetime.strptime(end, '%Y-%m-%d')

    #  TMIN、TAVG & TMAX
    results = session.query(func.min(Measurement.tobs),
                            func.avg(Measurement.tobs),
                            func.max(Measurement.tobs)).\
        filter(Measurement.date >= start_date).\
        filter(Measurement.date <= end_date).all()

    # Results
    temp_data = {
        "TMIN": results[0][0],
        "TAVG": results[0][1],
        "TMAX": results[0][2]
    }

    return jsonify(temp_data)

# Run Flask
if __name__ == '__main__':
    app.run(debug=True)
