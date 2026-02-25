from __future__ import annotations

import enum
import uuid
from datetime import datetime
from typing import Optional, List, Dict, Any

from sqlalchemy import (
    String, Boolean, DateTime, Integer, Float, ForeignKey, Text,
    Enum as SAEnum, Index, UniqueConstraint
)
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


# ==============================
# Base
# ==============================

class Base(DeclarativeBase):
    pass


def _uuid() -> uuid.UUID:
    return uuid.uuid4()


# ==============================
# Enums (equivalent to Prisma enums)
# ==============================

class UserRole(str, enum.Enum):
    SUPER_ADMIN = "SUPER_ADMIN"
    CONTENT_ADMIN = "CONTENT_ADMIN"
    REVIEWER = "REVIEWER"
    CUSTOMER_ADMIN = "CUSTOMER_ADMIN"
    FIELD_USER = "FIELD_USER"


class SolutionStatus(str, enum.Enum):
    DRAFT = "DRAFT"
    IN_REVIEW = "IN_REVIEW"
    PUBLISHED = "PUBLISHED"
    ARCHIVED = "ARCHIVED"


class Discipline(str, enum.Enum):
    FIRE = "FIRE"


class ScenarioType(str, enum.Enum):
    PENETRATION_SINGLE = "PENETRATION_SINGLE"
    PENETRATION_MIXED = "PENETRATION_MIXED"
    VENT_DUCT = "VENT_DUCT"
    LINEAR_JOINT = "LINEAR_JOINT"
    BOARD_ENCASEMENT = "BOARD_ENCASEMENT"


class ShapeType(str, enum.Enum):
    ROUND = "ROUND"
    RECTANGULAR = "RECTANGULAR"
    MIXED = "MIXED"
    JOINT = "JOINT"
    BOARD = "BOARD"


class InstallationSide(str, enum.Enum):
    ONE_SIDE = "ONE_SIDE"
    BOTH_SIDES = "BOTH_SIDES"
    NOT_APPLICABLE = "NOT_APPLICABLE"


# ==============================
# Users & Tenancy
# ==============================

class Organization(Base):
    __tablename__ = "organizations"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)

    created_by_id: Mapped[Optional[uuid.UUID]] = mapped_column(UUID(as_uuid=True), nullable=True)

    users: Mapped[List["User"]] = relationship(back_populates="organization", cascade="all, delete-orphan")


class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)
    email: Mapped[str] = mapped_column(String(320), unique=True, index=True, nullable=False)
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)

    role: Mapped[UserRole] = mapped_column(SAEnum(UserRole, name="user_role"), nullable=False)

    organization_id: Mapped[Optional[uuid.UUID]] = mapped_column(UUID(as_uuid=True), ForeignKey("organizations.id"), nullable=True)
    organization: Mapped[Optional[Organization]] = relationship(back_populates="users")

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)

    audit_logs: Mapped[List["AuditLog"]] = relationship(back_populates="user", cascade="all, delete-orphan")


# ==============================
# Manufacturers (data entity only)
# ==============================

class Manufacturer(Base):
    __tablename__ = "manufacturers"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)
    name: Mapped[str] = mapped_column(String(255), nullable=False, unique=True)
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)

    products: Mapped[List["Product"]] = relationship(back_populates="manufacturer", cascade="all, delete-orphan")
    solutions: Mapped[List["Solution"]] = relationship(back_populates="manufacturer", cascade="all, delete-orphan")


# ==============================
# Products
# ==============================

class Product(Base):
    __tablename__ = "products"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    manufacturer_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("manufacturers.id"), index=True, nullable=False)
    manufacturer: Mapped[Manufacturer] = relationship(back_populates="products")

    name: Mapped[str] = mapped_column(String(255), nullable=False)
    category: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
    technical_specs: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)

    solution_materials: Mapped[List["SolutionMaterial"]] = relationship(back_populates="product")


# ==============================
# Solutions (draft containers) + Versions (immutable)
# ==============================

class Solution(Base):
    __tablename__ = "solutions"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    manufacturer_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("manufacturers.id"), index=True, nullable=False)
    manufacturer: Mapped[Manufacturer] = relationship(back_populates="solutions")

    title: Mapped[str] = mapped_column(String(255), nullable=False)

    discipline: Mapped[Discipline] = mapped_column(SAEnum(Discipline, name="discipline"), nullable=False, default=Discipline.FIRE)
    scenario_type: Mapped[ScenarioType] = mapped_column(SAEnum(ScenarioType, name="scenario_type"), index=True, nullable=False)
    shape: Mapped[ShapeType] = mapped_column(SAEnum(ShapeType, name="shape_type"), index=True, nullable=False)

    status: Mapped[SolutionStatus] = mapped_column(SAEnum(SolutionStatus, name="solution_status"), index=True, nullable=False, default=SolutionStatus.DRAFT)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)

    versions: Mapped[List["SolutionVersion"]] = relationship(back_populates="solution", cascade="all, delete-orphan")


class SolutionVersion(Base):
    __tablename__ = "solution_versions"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    solution_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("solutions.id"), index=True, nullable=False)
    solution: Mapped[Solution] = relationship(back_populates="versions")

    version_number: Mapped[int] = mapped_column(Integer, nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=False, index=True, nullable=False)
    published_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True), nullable=True)

    ssr_template: Mapped[str] = mapped_column(String(512), nullable=False)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)

    applicability: Mapped[Optional["SolutionApplicability"]] = relationship(
        back_populates="solution_version", uselist=False, cascade="all, delete-orphan"
    )
    rules: Mapped[Optional["SolutionRule"]] = relationship(
        back_populates="solution_version", uselist=False, cascade="all, delete-orphan"
    )
    materials: Mapped[List["SolutionMaterial"]] = relationship(back_populates="solution_version", cascade="all, delete-orphan")
    steps: Mapped[List["SolutionStep"]] = relationship(back_populates="solution_version", cascade="all, delete-orphan")
    references: Mapped[List["SolutionReference"]] = relationship(back_populates="solution_version", cascade="all, delete-orphan")

    __table_args__ = (
        UniqueConstraint("solution_id", "version_number", name="uq_solution_version_number"),
        # Optional: enforce only one active version per solution in app logic or partial index (see note below).
    )


# ==============================
# Applicability (filters)
# ==============================

class SolutionApplicability(Base):
    __tablename__ = "solution_applicability"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    solution_version_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("solution_versions.id"), unique=True, index=True, nullable=False
    )
    solution_version: Mapped[SolutionVersion] = relationship(back_populates="applicability")

    # Stored as JSON arrays for flexibility
    construction_types: Mapped[Dict[str, Any]] = mapped_column(JSONB, nullable=False)  # e.g. ["MASSIVE_WALL","GYPSUM_WALL"]
    fire_ratings: Mapped[Dict[str, Any]] = mapped_column(JSONB, nullable=False)        # e.g. ["EI60","EI90"]
    service_types: Mapped[Dict[str, Any]] = mapped_column(JSONB, nullable=False)       # e.g. ["VENT","PIPE_METAL"]

    diameter_min: Mapped[Optional[float]] = mapped_column(Float, nullable=True)
    diameter_max: Mapped[Optional[float]] = mapped_column(Float, nullable=True)

    width_min: Mapped[Optional[float]] = mapped_column(Float, nullable=True)
    width_max: Mapped[Optional[float]] = mapped_column(Float, nullable=True)

    height_min: Mapped[Optional[float]] = mapped_column(Float, nullable=True)
    height_max: Mapped[Optional[float]] = mapped_column(Float, nullable=True)

    installation_sides: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)  # e.g. ["ONE_SIDE","BOTH_SIDES"]


# ==============================
# Rules (DSL JSON blobs)
# ==============================

class SolutionRule(Base):
    __tablename__ = "solution_rules"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    solution_version_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("solution_versions.id"), unique=True, index=True, nullable=False
    )
    solution_version: Mapped[SolutionVersion] = relationship(back_populates="rules")

    hole_formula: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)
    tolerance_rule: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)
    clearance_rule: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)
    insulation_rule: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)
    custom_constraints: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)


# ==============================
# Materials / Steps / References
# ==============================

class SolutionMaterial(Base):
    __tablename__ = "solution_materials"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    solution_version_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("solution_versions.id"), index=True, nullable=False
    )
    solution_version: Mapped[SolutionVersion] = relationship(back_populates="materials")

    product_id: Mapped[Optional[uuid.UUID]] = mapped_column(UUID(as_uuid=True), ForeignKey("products.id"), nullable=True)
    product: Mapped[Optional[Product]] = relationship(back_populates="solution_materials")

    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    quantity_logic: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)


class SolutionStep(Base):
    __tablename__ = "solution_steps"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    solution_version_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("solution_versions.id"), index=True, nullable=False
    )
    solution_version: Mapped[SolutionVersion] = relationship(back_populates="steps")

    step_number: Mapped[int] = mapped_column(Integer, nullable=False)
    instruction_text: Mapped[str] = mapped_column(Text, nullable=False)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)

    __table_args__ = (
        UniqueConstraint("solution_version_id", "step_number", name="uq_solution_step_order"),
    )


class SolutionReference(Base):
    __tablename__ = "solution_references"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    solution_version_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("solution_versions.id"), index=True, nullable=False
    )
    solution_version: Mapped[SolutionVersion] = relationship(back_populates="references")

    reference_code: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
    document_url: Mapped[Optional[str]] = mapped_column(String(1024), nullable=True)
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)


# ==============================
# Audit logs
# ==============================

class AuditLog(Base):
    __tablename__ = "audit_logs"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=_uuid)

    user_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"), index=True, nullable=False)
    user: Mapped[User] = relationship(back_populates="audit_logs")

    action: Mapped[str] = mapped_column(String(100), nullable=False)         # e.g. "SOLUTION_PUBLISHED"
    entity_type: Mapped[str] = mapped_column(String(100), nullable=False)    # e.g. "SolutionVersion"
    entity_id: Mapped[str] = mapped_column(String(64), nullable=False)       # store UUID as string for flexibility
    metadata: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSONB, nullable=True)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, nullable=False)


# ==============================
# Helpful Indexes (performance)
# ==============================

Index("ix_solution_versions_solution_id_active", SolutionVersion.solution_id, SolutionVersion.is_active)
Index("ix_solutions_scenario_shape", Solution.scenario_type, Solution.shape)
Index("ix_products_manufacturer", Product.manufacturer_id)