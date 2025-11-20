from flask import Flask, render_template, request
from pymongo import MongoClient
from bson.objectid import ObjectId
from config import MONGO_CONFIG

app = Flask(__name__)

def get_db():
    client = MongoClient(host=MONGO_CONFIG["host"], port=MONGO_CONFIG["port"])
    db = client[MONGO_CONFIG["database"]]
    return db

@app.route('/patrons')
def patron_list():
    db = get_db()

    #  Filters
    query = request.args.get('q', '').strip()
    patron_id_search = request.args.get('patron_id', '').strip()
    selected_age_group = request.args.get('age_group', '').strip()
    selected_role = request.args.get('role', '').strip()
    selected_circulation_month = request.args.get('circulation_month', '').strip()
    selected_home_library = request.args.get('home_library', '').strip()

    # Sorting and pagination
    sort_field = request.args.get('sort', 'checkout_total')
    sort_order = int(request.args.get('order', 1))  # 1=asc, -1=desc
    page = int(request.args.get('page', 1))
    per_page = 20
    skip = (page - 1) * per_page

    # Search filter
    filter_ = {}
    if selected_age_group:
        filter_["age_group"] = selected_age_group
    if selected_role:
        filter_["role_name"] = selected_role
    if selected_circulation_month:
        filter_["circulation_active_month"] = selected_circulation_month
    if selected_home_library:
        filter_["home_library_definition"] = selected_home_library

    # Patron ID search
    all_patrons = list(db.Patron.find({}, {"_id": 1}))  # get all IDs
    matched_ids = []
    if patron_id_search:
        for p in all_patrons:
            if str(p["_id"])[-8:] == patron_id_search:
                matched_ids.append(p["_id"])
        filter_["_id"] = {"$in": matched_ids} if matched_ids else {"$in": []}

    if query:
        filter_["$or"] = [
            {"patron_type_definition": {"$regex": query, "$options": "i"}},
            {"age_group": {"$regex": query, "$options": "i"}},
            {"role_name": {"$regex": query, "$options": "i"}},
            {"home_library_definition": {"$regex": query, "$options": "i"}},
        ]

    # Projection
    projection = {
        "_id": 1,
        "patron_type_code": 1,
        "patron_type_definition": 1,
        "checkout_total": 1,
        "renewal_total": 1,
        "age_group": 1,
        "role_name": 1,
        "circulation_active_month": 1,
        "home_library_definition": 1
    }

    total_count = db.Patron.count_documents(filter_)
    rows = list(
        db.Patron.find(filter_, projection)
        .sort(sort_field, sort_order)
        .skip(skip)
        .limit(per_page)
    )

    # short_id for easy ID searching
    for r in rows:
        r["short_id"] = str(r["_id"])[-8:]

    total_pages = (total_count + per_page - 1) // per_page

    # Dropdown options
    age_groups = list(db.Patron.distinct("age_group"))
    roles = list(db.Patron.distinct("role_name"))
    home_libraries = list(db.Patron.distinct("home_library_definition"))

    # Circulation months
    all_months = list(db.Patron.distinct("circulation_active_month"))
    month_order = ["January", "February", "March", "April", "May", "June",
                   "July", "August", "September", "October", "November", "December"]
    circulation_months = [m for m in month_order if m in all_months and m.strip() != ""]

  # Render patron page with data and filters
    return render_template(
        'patron.html',
        rows=rows,
        query=query,
        patron_id_search=patron_id_search,
        selected_age_group=selected_age_group,
        selected_role=selected_role,
        selected_circulation_month=selected_circulation_month,
        selected_home_library=selected_home_library,
        age_groups=age_groups,
        roles=roles,
        circulation_months=circulation_months,
        home_libraries=home_libraries,
        sort_field=sort_field,
        sort_order=sort_order,
        page=page,
        total_pages=total_pages
    )

# Patron profile
@app.route('/patron/<short_id>')
def patron_profile(short_id):
    db = get_db()
    # Match last 8 chars
    all_patrons = list(db.Patron.find({}, {}))
    patron = None
    for p in all_patrons:
        if str(p["_id"])[-8:] == short_id:
            patron = p
            break

    if not patron:
        return f"Patron with ID ending {short_id} not found!", 404

    patron["short_id"] = str(patron["_id"])[-8:]
    return render_template('profile.html', patron=patron)

if __name__ == '__main__':
    app.run(debug=True)
